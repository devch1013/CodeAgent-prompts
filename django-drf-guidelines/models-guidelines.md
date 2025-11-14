# Django Models 가이드라인

## 개요
이 가이드라인은 Django 모델 작성 시 따라야 할 최신 트렌드와 베스트 프랙티스를 제공합니다.

## 기본 원칙

### 1. Type Hints 사용
```python
from typing import Optional
from django.db import models
from django.contrib.auth.models import User

class Product(models.Model):
    name: str = models.CharField(max_length=255)
    price: int = models.IntegerField()
    user: User = models.ForeignKey(User, on_delete=models.CASCADE)

    def calculate_discounted_price(self, discount_rate: float) -> float:
        return float(self.price) * (1 - discount_rate)
```

### 2. 모델 필드 정의 순서
```python
class Article(models.Model):
    # 1. Database fields (in logical order)
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()

    # 2. Relationship fields
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    category = models.ForeignKey('Category', on_delete=models.SET_NULL, null=True)
    tags = models.ManyToManyField('Tag', blank=True)

    # 3. Metadata fields
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    # 4. Meta class
    class Meta:
        ordering = ['-created_at']
        verbose_name = 'Article'
        verbose_name_plural = 'Articles'
        indexes = [
            models.Index(fields=['-created_at']),
            models.Index(fields=['slug']),
        ]

    # 5. Magic methods
    def __str__(self) -> str:
        return self.title

    # 6. Custom methods (simple utilities)
    def get_absolute_url(self) -> str:
        return f'/articles/{self.slug}/'
```

### 3. 비즈니스 로직은 Services로 이동
```python
# ❌ 나쁜 예 - 모델에 복잡한 비즈니스 로직
class Order(models.Model):
    def process_order(self):
        # Send email
        # Update inventory
        # Process payment
        # Create invoice
        pass

# ✅ 좋은 예 - 모델은 데이터와 간단한 유틸리티만
class Order(models.Model):
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

    def is_completed(self) -> bool:
        return self.status == OrderStatus.COMPLETED.value

    def can_be_cancelled(self) -> bool:
        return self.status in [OrderStatus.PENDING.value, OrderStatus.PROCESSING.value]

# 비즈니스 로직은 services/order_service.py로 이동
```

### 4. Custom Manager와 QuerySet 활용
```python
from django.db import models
from django.db.models import QuerySet, Q
from typing import Optional

class ArticleQuerySet(QuerySet):
    def published(self) -> QuerySet:
        return self.filter(is_published=True)

    def by_author(self, author_id: int) -> QuerySet:
        return self.filter(author_id=author_id)

    def search(self, query: str) -> QuerySet:
        return self.filter(
            Q(title__icontains=query) | Q(content__icontains=query)
        )

class ArticleManager(models.Manager):
    def get_queryset(self) -> ArticleQuerySet:
        return ArticleQuerySet(self.model, using=self._db)

    def published(self) -> QuerySet:
        return self.get_queryset().published()

    def by_author(self, author_id: int) -> QuerySet:
        return self.get_queryset().by_author(author_id)

class Article(models.Model):
    title = models.CharField(max_length=200)
    is_published = models.BooleanField(default=False)
    author = models.ForeignKey(User, on_delete=models.CASCADE)

    objects = ArticleManager()
```

### 5. 적절한 related_name 사용
```python
class Author(models.Model):
    name = models.CharField(max_length=100)

class Book(models.Model):
    title = models.CharField(max_length=200)
    # ✅ 명확하고 직관적인 related_name
    author = models.ForeignKey(
        Author,
        on_delete=models.CASCADE,
        related_name='books'  # author.books.all()
    )

class Comment(models.Model):
    content = models.TextField()
    article = models.ForeignKey(
        'Article',
        on_delete=models.CASCADE,
        related_name='comments'  # article.comments.all()
    )
```

### 6. Abstract Base Models 활용
```python
from django.db import models

class TimeStampedModel(models.Model):
    """공통 타임스탬프 필드를 제공하는 추상 모델"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

class SoftDeleteModel(models.Model):
    """Soft delete 기능을 제공하는 추상 모델"""
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

class Article(TimeStampedModel, SoftDeleteModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
```

### 7. Database 제약조건 활용
```python
from django.db import models
from django.core.validators import MinValueValidator, MaxValueValidator

class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0)]
    )
    stock = models.IntegerField(
        validators=[MinValueValidator(0)]
    )

    class Meta:
        constraints = [
            models.CheckConstraint(
                check=models.Q(price__gte=0),
                name='price_non_negative'
            ),
            models.CheckConstraint(
                check=models.Q(stock__gte=0),
                name='stock_non_negative'
            ),
            models.UniqueConstraint(
                fields=['name', 'category'],
                name='unique_product_per_category'
            )
        ]
```

### 8. Select/Prefetch Related 최적화
```python
# models.py에 최적화된 쿼리 메서드 추가
class ArticleQuerySet(QuerySet):
    def with_related(self) -> QuerySet:
        """필요한 관계를 미리 로드"""
        return self.select_related(
            'author',
            'category'
        ).prefetch_related(
            'tags',
            'comments__author'
        )

    def with_counts(self) -> QuerySet:
        """카운트를 미리 계산"""
        from django.db.models import Count
        return self.annotate(
            comment_count=Count('comments'),
            like_count=Count('likes')
        )
```

### 9. Enum을 활용한 상태 관리
```python
# enums.py와 함께 사용
from apps.orders.enums import OrderStatus

class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices(),
        default=OrderStatus.PENDING.value
    )

    def is_status(self, status: OrderStatus) -> bool:
        return self.status == status.value
```

### 10. JSONField 활용 (PostgreSQL)
```python
from django.db import models
from typing import Dict, Any

class Product(models.Model):
    name = models.CharField(max_length=255)
    # 유연한 메타데이터 저장
    metadata = models.JSONField(default=dict, blank=True)
    # 다국어 지원
    translations = models.JSONField(default=dict, blank=True)

    def get_metadata(self, key: str, default: Any = None) -> Any:
        return self.metadata.get(key, default)

    def set_metadata(self, key: str, value: Any) -> None:
        self.metadata[key] = value
        self.save(update_fields=['metadata'])
```

## 주의사항

### ❌ 피해야 할 패턴

1. **모델에 과도한 비즈니스 로직 포함**
```python
# ❌ 나쁜 예
class Order(models.Model):
    def send_confirmation_email(self):  # 이메일 발송은 service로
        pass

    def charge_payment(self):  # 결제 처리는 service로
        pass
```

2. **N+1 쿼리 발생**
```python
# ❌ 나쁜 예
articles = Article.objects.all()
for article in articles:
    print(article.author.name)  # N+1 쿼리 발생

# ✅ 좋은 예
articles = Article.objects.select_related('author')
for article in articles:
    print(article.author.name)
```

3. **불명확한 필드명**
```python
# ❌ 나쁜 예
class User(models.Model):
    flag = models.BooleanField()  # 무슨 flag?
    status = models.IntegerField()  # 어떤 상태?

# ✅ 좋은 예
class User(models.Model):
    is_email_verified = models.BooleanField(default=False)
    account_status = models.CharField(max_length=20, choices=AccountStatus.choices())
```

## 체크리스트

- [ ] Type hints를 모든 메서드에 추가했는가?
- [ ] 복잡한 비즈니스 로직을 services로 분리했는가?
- [ ] Custom Manager/QuerySet을 활용했는가?
- [ ] related_name을 명확하게 설정했는가?
- [ ] Database 제약조건을 추가했는가?
- [ ] 필드 순서가 논리적인가?
- [ ] Meta 클래스에 적절한 인덱스를 추가했는가?
- [ ] N+1 쿼리를 방지하는 메서드를 제공했는가?
- [ ] Enum을 사용하여 상태 값을 관리했는가?
- [ ] Abstract base model로 공통 필드를 재사용했는가?
