# Django Test Factories 가이드라인

## 개요
Factory Pattern을 사용하여 테스트 데이터를 쉽고 일관되게 생성합니다. Factory Boy 라이브러리를 활용합니다.

## 설치

```bash
pip install factory-boy faker
```

## 기본 원칙

### 1. 기본 Factory 정의
```python
# factories.py
import factory
from factory.django import DjangoModelFactory
from factory import Faker, SubFactory, LazyAttribute
from django.contrib.auth.models import User
from apps.articles.models import Article, Comment, Tag

class UserFactory(DjangoModelFactory):
    """User 모델 Factory"""
    class Meta:
        model = User

    username = Faker('user_name')
    email = Faker('email')
    first_name = Faker('first_name')
    last_name = Faker('last_name')
    is_active = True
    is_staff = False

    @factory.post_generation
    def password(self, create, extracted, **kwargs):
        """비밀번호 설정"""
        if not create:
            return

        if extracted:
            self.set_password(extracted)
        else:
            self.set_password('defaultpassword123')

# 사용 예시
user = UserFactory()
user_with_password = UserFactory(password='mypassword')
admin_user = UserFactory(is_staff=True, is_superuser=True)
```

### 2. 관계가 있는 Factory
```python
# factories.py
class TagFactory(DjangoModelFactory):
    """Tag 모델 Factory"""
    class Meta:
        model = Tag

    name = Faker('word')
    slug = LazyAttribute(lambda obj: obj.name.lower())

class ArticleFactory(DjangoModelFactory):
    """Article 모델 Factory"""
    class Meta:
        model = Article

    title = Faker('sentence', nb_words=6)
    content = Faker('paragraph', nb_sentences=10)
    author = SubFactory(UserFactory)
    is_published = False

    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        """Many-to-Many 관계 처리"""
        if not create:
            return

        if extracted:
            for tag in extracted:
                self.tags.add(tag)

class CommentFactory(DjangoModelFactory):
    """Comment 모델 Factory"""
    class Meta:
        model = Comment

    content = Faker('paragraph', nb_sentences=3)
    article = SubFactory(ArticleFactory)
    author = SubFactory(UserFactory)

# 사용 예시
article = ArticleFactory()
article_with_tags = ArticleFactory(tags=[TagFactory(), TagFactory()])
comment = CommentFactory()
comment_on_specific_article = CommentFactory(article=article)
```

### 3. Trait를 활용한 변형
```python
# factories.py
class ArticleFactory(DjangoModelFactory):
    """Article 모델 Factory"""
    class Meta:
        model = Article

    title = Faker('sentence', nb_words=6)
    content = Faker('paragraph', nb_sentences=10)
    author = SubFactory(UserFactory)
    is_published = False

    class Params:
        # Trait 정의
        published = factory.Trait(
            is_published=True,
            published_at=factory.Faker('date_time_this_year')
        )

        with_tags = factory.Trait(
            tags=factory.RelatedFactoryList(
                TagFactory,
                factory_related_name='articles',
                size=3
            )
        )

        popular = factory.Trait(
            is_published=True,
            view_count=factory.Faker('random_int', min=1000, max=10000),
            like_count=factory.Faker('random_int', min=100, max=1000)
        )

# 사용 예시
draft_article = ArticleFactory()
published_article = ArticleFactory(published=True)
popular_article = ArticleFactory(published=True, popular=True)
```

### 4. Sequence 활용
```python
# factories.py
class UserFactory(DjangoModelFactory):
    class Meta:
        model = User

    # Sequence를 사용한 유일한 값 생성
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.Sequence(lambda n: f'user{n}@example.com')

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Sequence(lambda n: f'Article {n}')
    slug = factory.Sequence(lambda n: f'article-{n}')

# 사용 예시
user1 = UserFactory()  # username: 'user0', email: 'user0@example.com'
user2 = UserFactory()  # username: 'user1', email: 'user1@example.com'
```

### 5. LazyAttribute와 LazyFunction
```python
# factories.py
from datetime import datetime, timedelta

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = Faker('sentence')
    # LazyAttribute: 객체의 다른 속성 참조
    slug = LazyAttribute(lambda obj: obj.title.lower().replace(' ', '-'))

    # LazyFunction: 함수 결과 사용
    created_at = factory.LazyFunction(datetime.now)
    published_at = factory.LazyFunction(
        lambda: datetime.now() + timedelta(days=1)
    )

class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    order_number = factory.LazyFunction(
        lambda: f'ORD-{datetime.now().strftime("%Y%m%d")}-{factory.Faker("random_int", min=1000, max=9999)}'
    )
```

### 6. Faker Provider 커스터마이징
```python
# factories.py
from faker import Faker as FakerClass
from faker.providers import BaseProvider

# 커스텀 Provider
class CustomProvider(BaseProvider):
    def korean_name(self):
        """한국 이름 생성"""
        surnames = ['김', '이', '박', '최', '정']
        names = ['민수', '지영', '철수', '영희', '현우']
        return f'{self.random_element(surnames)}{self.random_element(names)}'

    def korean_phone_number(self):
        """한국 전화번호 생성"""
        return f'010-{self.random_int(1000, 9999)}-{self.random_int(1000, 9999)}'

# Provider 등록
fake = FakerClass('ko_KR')
fake.add_provider(CustomProvider)

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Faker('user_name')
    # 커스텀 Provider 사용
    full_name = factory.Faker('korean_name', locale='ko_KR')
    phone_number = factory.Faker('korean_phone_number', locale='ko_KR')
```

### 7. SubFactory vs RelatedFactory
```python
# factories.py

# SubFactory: ForeignKey 관계
class CommentFactory(DjangoModelFactory):
    class Meta:
        model = Comment

    content = Faker('paragraph')
    article = SubFactory(ArticleFactory)  # Comment를 만들 때 Article도 생성

# RelatedFactory: 역방향 관계 (reverse)
class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = Faker('sentence')
    author = SubFactory(UserFactory)

    # Article 생성 시 자동으로 Comment 3개 생성
    comment1 = factory.RelatedFactory(
        CommentFactory,
        factory_related_name='article'
    )
    comment2 = factory.RelatedFactory(
        CommentFactory,
        factory_related_name='article'
    )
    comment3 = factory.RelatedFactory(
        CommentFactory,
        factory_related_name='article'
    )

# 또는 List 버전
class ArticleWithCommentsFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = Faker('sentence')

    @factory.post_generation
    def comments(self, create, extracted, **kwargs):
        """여러 댓글 생성"""
        if not create:
            return

        count = extracted or 3
        CommentFactory.create_batch(count, article=self)

# 사용 예시
article = ArticleWithCommentsFactory()  # 댓글 3개와 함께
article = ArticleWithCommentsFactory(comments=5)  # 댓글 5개와 함께
```

### 8. 여러 객체 생성 (Batch)
```python
# 테스트에서 사용
def test_article_list():
    # 여러 개 한번에 생성
    articles = ArticleFactory.create_batch(10)
    assert len(articles) == 10

    # 특정 속성으로 여러 개 생성
    published_articles = ArticleFactory.create_batch(5, is_published=True)
    assert all(article.is_published for article in published_articles)

    # 다양한 속성으로 생성
    users = UserFactory.create_batch(
        3,
        is_staff=factory.Iterator([True, False, True])
    )
```

### 9. 복잡한 데이터 생성
```python
# factories.py
class OrderItemFactory(DjangoModelFactory):
    class Meta:
        model = OrderItem

    order = SubFactory('apps.orders.factories.OrderFactory')
    product = SubFactory('apps.products.factories.ProductFactory')
    quantity = Faker('random_int', min=1, max=10)
    unit_price = LazyAttribute(lambda obj: obj.product.price)
    total_price = LazyAttribute(lambda obj: obj.quantity * obj.unit_price)

class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    user = SubFactory(UserFactory)
    status = OrderStatus.PENDING.value
    total_amount = 0

    @factory.post_generation
    def items(self, create, extracted, **kwargs):
        """주문 아이템 생성 및 총액 계산"""
        if not create:
            return

        if extracted:
            items = extracted
        else:
            # 기본적으로 3개의 아이템 생성
            items = OrderItemFactory.create_batch(3, order=self)

        # 총액 계산
        self.total_amount = sum(item.total_price for item in items)
        self.save()

# 사용 예시
order = OrderFactory()  # 3개의 아이템과 함께 생성
order = OrderFactory(items=OrderItemFactory.create_batch(5, order=None))  # 5개 아이템
```

### 10. 조건부 생성
```python
# factories.py
class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = Faker('sentence')
    content = Faker('paragraph')
    author = SubFactory(UserFactory)

    # 조건부 필드
    published_at = factory.Maybe(
        'is_published',
        yes_declaration=factory.Faker('date_time_this_year'),
        no_declaration=None
    )

    # 조건부 관계
    @factory.post_generation
    def setup_published(self, create, extracted, **kwargs):
        """발행된 게시글인 경우 추가 설정"""
        if not create:
            return

        if self.is_published:
            self.published_at = self.published_at or datetime.now()
            # 발행된 게시글에는 자동으로 태그 추가
            if not self.tags.exists():
                self.tags.set(TagFactory.create_batch(3))

# 사용 예시
draft = ArticleFactory(is_published=False)  # published_at is None
published = ArticleFactory(is_published=True)  # published_at is set, has tags
```

### 11. 실제 테스트에서 활용
```python
# tests/test_articles.py
import pytest
from apps.articles.factories import ArticleFactory, UserFactory, TagFactory

@pytest.mark.django_db
class TestArticle:
    def test_create_article(self):
        """게시글 생성 테스트"""
        article = ArticleFactory()
        assert article.id is not None
        assert article.author is not None

    def test_published_article(self):
        """발행된 게시글 테스트"""
        article = ArticleFactory(is_published=True)
        assert article.is_published is True
        assert article.published_at is not None

    def test_article_with_tags(self):
        """태그가 있는 게시글 테스트"""
        tags = TagFactory.create_batch(3)
        article = ArticleFactory(tags=tags)
        assert article.tags.count() == 3

    def test_article_list(self):
        """게시글 목록 테스트"""
        ArticleFactory.create_batch(10, is_published=True)
        ArticleFactory.create_batch(5, is_published=False)

        published_count = Article.objects.filter(is_published=True).count()
        assert published_count == 10

@pytest.mark.django_db
class TestArticleService:
    def test_create_article_service(self):
        """게시글 생성 서비스 테스트"""
        user = UserFactory()
        tags = TagFactory.create_batch(2)

        service = ArticleService()
        article = service.create_article(
            title='Test Article',
            content='Test Content',
            author=user,
            tag_ids=[tag.id for tag in tags]
        )

        assert article.title == 'Test Article'
        assert article.tags.count() == 2
```

### 12. Fixture와 Factory 조합
```python
# conftest.py
import pytest
from apps.users.factories import UserFactory
from apps.articles.factories import ArticleFactory

@pytest.fixture
def user():
    """기본 사용자"""
    return UserFactory()

@pytest.fixture
def admin_user():
    """관리자 사용자"""
    return UserFactory(is_staff=True, is_superuser=True)

@pytest.fixture
def published_articles():
    """발행된 게시글 10개"""
    return ArticleFactory.create_batch(10, is_published=True)

@pytest.fixture
def article_with_comments():
    """댓글이 있는 게시글"""
    article = ArticleFactory(is_published=True)
    CommentFactory.create_batch(5, article=article)
    return article

# tests/test_views.py
@pytest.mark.django_db
def test_article_list_view(client, published_articles):
    """게시글 목록 조회 테스트"""
    response = client.get('/api/articles/')
    assert response.status_code == 200
    assert len(response.data) == 10

@pytest.mark.django_db
def test_create_article_view(client, user):
    """게시글 생성 테스트"""
    client.force_authenticate(user=user)
    response = client.post('/api/articles/', {
        'title': 'New Article',
        'content': 'Content here'
    })
    assert response.status_code == 201
```

## 베스트 프랙티스

### 1. Factory 구조화
```python
# factories/__init__.py
from .user_factories import UserFactory, AdminFactory
from .article_factories import ArticleFactory, PublishedArticleFactory
from .order_factories import OrderFactory, OrderItemFactory

__all__ = [
    'UserFactory',
    'AdminFactory',
    'ArticleFactory',
    'PublishedArticleFactory',
    'OrderFactory',
    'OrderItemFactory',
]

# factories/user_factories.py
class UserFactory(DjangoModelFactory):
    """기본 사용자 Factory"""
    pass

class AdminFactory(UserFactory):
    """관리자 Factory"""
    is_staff = True
    is_superuser = True

# factories/article_factories.py
class ArticleFactory(DjangoModelFactory):
    """기본 게시글 Factory"""
    pass

class PublishedArticleFactory(ArticleFactory):
    """발행된 게시글 Factory"""
    is_published = True
    published_at = factory.LazyFunction(datetime.now)
```

### 2. 재사용 가능한 Trait
```python
# factories.py
class TimestampedFactoryMixin:
    """타임스탬프 Mixin"""
    created_at = factory.LazyFunction(datetime.now)
    updated_at = factory.LazyFunction(datetime.now)

class ArticleFactory(TimestampedFactoryMixin, DjangoModelFactory):
    class Meta:
        model = Article

    title = Faker('sentence')
```

## 주의사항

### ❌ 피해야 할 패턴

1. **테스트에서 직접 모델 생성**
```python
# ❌ 나쁜 예
def test_article():
    user = User.objects.create(username='test', email='test@example.com')
    article = Article.objects.create(title='Test', content='Content', author=user)

# ✅ 좋은 예
def test_article():
    article = ArticleFactory()
```

2. **모든 필드를 수동으로 설정**
```python
# ❌ 나쁜 예
article = ArticleFactory(
    title='Test',
    content='Content',
    author=UserFactory(),
    is_published=False,
    created_at=datetime.now()
)

# ✅ 좋은 예
article = ArticleFactory()  # 기본값 사용
article = ArticleFactory(is_published=True)  # 필요한 것만 override
```

## 체크리스트

- [ ] Factory Boy를 설치했는가?
- [ ] 모든 모델에 대한 Factory를 작성했는가?
- [ ] Faker를 활용하여 현실적인 데이터를 생성하는가?
- [ ] 관계(ForeignKey, ManyToMany)를 적절히 처리했는가?
- [ ] Trait를 활용하여 변형을 만들었는가?
- [ ] 테스트에서 Factory를 일관되게 사용하는가?
- [ ] Factory를 도메인별로 구조화했는가?
- [ ] post_generation을 활용하여 복잡한 로직을 처리했는가?
- [ ] Sequence를 사용하여 유일한 값을 생성하는가?
- [ ] Fixture와 Factory를 함께 활용하는가?
