# Django Services 가이드라인

## 개요
Service Layer는 비즈니스 로직을 캡슐화하고 Views, Models, Serializers로부터 분리하는 패턴입니다. 이는 2024-2025 Django DRF의 핵심 트렌드입니다.

## Service Layer의 목적

1. **관심사의 분리 (Separation of Concerns)**
   - Views: HTTP 요청/응답 처리
   - Models: 데이터 구조 및 간단한 유틸리티
   - Services: 비즈니스 로직

2. **재사용성**
   - 여러 Views, Tasks, Management Commands에서 동일한 로직 사용

3. **테스트 용이성**
   - HTTP 오버헤드 없이 순수 비즈니스 로직 테스트

## 기본 구조

### 1. 기본 Service 클래스
```python
# services/base.py
from typing import TypeVar, Generic, Type, Optional, List, Dict, Any
from django.db import models, transaction
from django.core.exceptions import ValidationError

ModelType = TypeVar('ModelType', bound=models.Model)

class BaseService(Generic[ModelType]):
    """모든 Service의 기본 클래스"""
    model: Type[ModelType]

    def __init__(self, user: Optional[Any] = None):
        self.user = user

    def get_queryset(self):
        """기본 쿼리셋 반환"""
        return self.model.objects.all()

    def get_by_id(self, id: int) -> Optional[ModelType]:
        """ID로 객체 조회"""
        try:
            return self.get_queryset().get(id=id)
        except self.model.DoesNotExist:
            return None

    def filter(self, **kwargs) -> List[ModelType]:
        """필터링"""
        return list(self.get_queryset().filter(**kwargs))

    @transaction.atomic
    def create(self, **data) -> ModelType:
        """객체 생성"""
        instance = self.model(**data)
        instance.full_clean()
        instance.save()
        return instance

    @transaction.atomic
    def update(self, instance: ModelType, **data) -> ModelType:
        """객체 수정"""
        for key, value in data.items():
            setattr(instance, key, value)
        instance.full_clean()
        instance.save()
        return instance

    @transaction.atomic
    def delete(self, instance: ModelType) -> None:
        """객체 삭제"""
        instance.delete()
```

### 2. 구체적인 Service 구현
```python
# services/article_service.py
from typing import List, Optional, Dict, Any
from django.db import transaction
from django.contrib.auth.models import User
from apps.articles.models import Article, Tag
from apps.articles.services.base import BaseService
from apps.articles.exceptions import ArticlePermissionDenied, ArticleNotFound

class ArticleService(BaseService[Article]):
    model = Article

    def get_queryset(self):
        """최적화된 쿼리셋 반환"""
        return super().get_queryset().select_related(
            'author',
            'category'
        ).prefetch_related('tags')

    def get_published_articles(self) -> List[Article]:
        """공개된 게시글만 조회"""
        return list(
            self.get_queryset()
            .filter(is_published=True)
            .order_by('-created_at')
        )

    def get_user_articles(self, user_id: int) -> List[Article]:
        """특정 사용자의 게시글 조회"""
        return list(
            self.get_queryset()
            .filter(author_id=user_id)
            .order_by('-created_at')
        )

    @transaction.atomic
    def create_article(
        self,
        title: str,
        content: str,
        author: User,
        category_id: Optional[int] = None,
        tag_ids: Optional[List[int]] = None
    ) -> Article:
        """게시글 생성"""
        # 비즈니스 로직: 제목 중복 검사
        if Article.objects.filter(title__iexact=title).exists():
            raise ValidationError("Article with this title already exists")

        # 게시글 생성
        article = Article.objects.create(
            title=title,
            content=content,
            author=author,
            category_id=category_id
        )

        # 태그 연결
        if tag_ids:
            article.tags.set(tag_ids)

        return article

    @transaction.atomic
    def update_article(
        self,
        article_id: int,
        user: User,
        **update_data
    ) -> Article:
        """게시글 수정"""
        article = self.get_by_id(article_id)
        if not article:
            raise ArticleNotFound(f"Article with id {article_id} not found")

        # 권한 검증
        if article.author != user:
            raise ArticlePermissionDenied("You don't have permission to update this article")

        # 태그 처리
        tag_ids = update_data.pop('tag_ids', None)

        # 게시글 수정
        for key, value in update_data.items():
            setattr(article, key, value)
        article.save()

        # 태그 업데이트
        if tag_ids is not None:
            article.tags.set(tag_ids)

        return article

    @transaction.atomic
    def publish_article(self, article_id: int, user: User) -> Article:
        """게시글 발행"""
        article = self.get_by_id(article_id)
        if not article:
            raise ArticleNotFound(f"Article with id {article_id} not found")

        if article.author != user:
            raise ArticlePermissionDenied("You don't have permission to publish this article")

        if article.is_published:
            raise ValidationError("Article is already published")

        article.is_published = True
        article.save(update_fields=['is_published'])

        # 추가 비즈니스 로직 (알림 발송 등)
        self._notify_subscribers(article)

        return article

    @transaction.atomic
    def delete_article(self, article_id: int, user: User) -> None:
        """게시글 삭제"""
        article = self.get_by_id(article_id)
        if not article:
            raise ArticleNotFound(f"Article with id {article_id} not found")

        if article.author != user and not user.is_staff:
            raise ArticlePermissionDenied("You don't have permission to delete this article")

        article.delete()

    def _notify_subscribers(self, article: Article) -> None:
        """구독자에게 알림 발송 (private method)"""
        # 알림 발송 로직
        pass
```

### 3. 복잡한 비즈니스 로직 예시
```python
# services/order_service.py
from typing import List, Dict, Any
from decimal import Decimal
from django.db import transaction
from django.utils import timezone
from apps.orders.models import Order, OrderItem
from apps.products.models import Product
from apps.users.models import User
from apps.orders.enums import OrderStatus
from apps.orders.exceptions import InsufficientStockError, InvalidOrderError

class OrderService(BaseService[Order]):
    model = Order

    @transaction.atomic
    def create_order(
        self,
        user: User,
        items: List[Dict[str, Any]],  # [{"product_id": 1, "quantity": 2}, ...]
        shipping_address: str,
        payment_method: str
    ) -> Order:
        """주문 생성"""
        # 1. 주문 생성
        order = Order.objects.create(
            user=user,
            status=OrderStatus.PENDING.value,
            shipping_address=shipping_address,
            payment_method=payment_method,
            total_amount=Decimal('0.00')
        )

        total_amount = Decimal('0.00')

        # 2. 주문 아이템 생성 및 재고 확인
        for item_data in items:
            product = Product.objects.select_for_update().get(
                id=item_data['product_id']
            )
            quantity = item_data['quantity']

            # 재고 확인
            if product.stock < quantity:
                raise InsufficientStockError(
                    f"Insufficient stock for product {product.name}"
                )

            # 주문 아이템 생성
            item_total = product.price * quantity
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=quantity,
                unit_price=product.price,
                total_price=item_total
            )

            # 재고 차감
            product.stock -= quantity
            product.save(update_fields=['stock'])

            total_amount += item_total

        # 3. 총액 업데이트
        order.total_amount = total_amount
        order.save(update_fields=['total_amount'])

        return order

    @transaction.atomic
    def process_payment(self, order_id: int) -> Order:
        """결제 처리"""
        order = self.get_by_id(order_id)
        if not order:
            raise InvalidOrderError(f"Order {order_id} not found")

        if order.status != OrderStatus.PENDING.value:
            raise InvalidOrderError(f"Order {order_id} is not in pending status")

        # 외부 결제 시스템 호출 (여기서는 생략)
        payment_result = self._process_external_payment(order)

        if payment_result['success']:
            order.status = OrderStatus.PAID.value
            order.paid_at = timezone.now()
            order.save(update_fields=['status', 'paid_at'])

            # 결제 완료 후 처리
            self._send_order_confirmation_email(order)
            self._notify_warehouse(order)
        else:
            order.status = OrderStatus.PAYMENT_FAILED.value
            order.save(update_fields=['status'])

        return order

    @transaction.atomic
    def cancel_order(self, order_id: int, user: User, reason: str) -> Order:
        """주문 취소"""
        order = self.get_by_id(order_id)
        if not order:
            raise InvalidOrderError(f"Order {order_id} not found")

        if order.user != user:
            raise PermissionError("You don't have permission to cancel this order")

        if order.status not in [OrderStatus.PENDING.value, OrderStatus.PAID.value]:
            raise InvalidOrderError("This order cannot be cancelled")

        # 재고 복원
        for item in order.items.select_related('product'):
            product = item.product
            product.stock += item.quantity
            product.save(update_fields=['stock'])

        # 주문 상태 변경
        order.status = OrderStatus.CANCELLED.value
        order.cancelled_at = timezone.now()
        order.cancellation_reason = reason
        order.save(update_fields=['status', 'cancelled_at', 'cancellation_reason'])

        # 환불 처리
        if order.paid_at:
            self._process_refund(order)

        return order

    def _process_external_payment(self, order: Order) -> Dict[str, Any]:
        """외부 결제 시스템 호출"""
        # 실제 결제 API 호출
        return {'success': True, 'transaction_id': 'xxx'}

    def _send_order_confirmation_email(self, order: Order) -> None:
        """주문 확인 이메일 발송"""
        pass

    def _notify_warehouse(self, order: Order) -> None:
        """창고에 배송 준비 알림"""
        pass

    def _process_refund(self, order: Order) -> None:
        """환불 처리"""
        pass
```

### 4. Service 조합 (Service Composition)
```python
# services/blog_service.py
from typing import List
from django.db import transaction
from apps.articles.services.article_service import ArticleService
from apps.comments.services.comment_service import CommentService
from apps.notifications.services.notification_service import NotificationService

class BlogService:
    """여러 Service를 조합하여 복잡한 비즈니스 플로우 처리"""

    def __init__(self, user=None):
        self.article_service = ArticleService(user=user)
        self.comment_service = CommentService(user=user)
        self.notification_service = NotificationService(user=user)

    @transaction.atomic
    def create_article_with_notifications(
        self,
        title: str,
        content: str,
        author,
        tag_ids: List[int]
    ):
        """게시글 작성 및 구독자 알림"""
        # 1. 게시글 생성
        article = self.article_service.create_article(
            title=title,
            content=content,
            author=author,
            tag_ids=tag_ids
        )

        # 2. 구독자에게 알림
        subscribers = author.followers.all()
        for subscriber in subscribers:
            self.notification_service.create_notification(
                user=subscriber,
                message=f"{author.username} posted a new article: {title}",
                related_object=article
            )

        return article

    @transaction.atomic
    def add_comment_with_notification(
        self,
        article_id: int,
        user,
        content: str
    ):
        """댓글 작성 및 게시글 작성자에게 알림"""
        # 1. 댓글 생성
        comment = self.comment_service.create_comment(
            article_id=article_id,
            user=user,
            content=content
        )

        # 2. 게시글 작성자에게 알림
        article = comment.article
        if article.author != user:
            self.notification_service.create_notification(
                user=article.author,
                message=f"{user.username} commented on your article",
                related_object=comment
            )

        return comment
```

### 5. 비동기 Service (Async Support)
```python
# services/async_article_service.py
from typing import List, Optional
from asgiref.sync import sync_to_async
from apps.articles.models import Article

class AsyncArticleService:
    """비동기 작업을 위한 Service"""

    @sync_to_async
    def get_published_articles(self) -> List[Article]:
        """비동기로 공개 게시글 조회"""
        return list(
            Article.objects
            .filter(is_published=True)
            .select_related('author')
            .order_by('-created_at')
        )

    @sync_to_async
    def get_article_by_id(self, article_id: int) -> Optional[Article]:
        """비동기로 게시글 조회"""
        try:
            return Article.objects.select_related('author').get(id=article_id)
        except Article.DoesNotExist:
            return None

    async def get_article_with_related_data(self, article_id: int):
        """여러 비동기 작업 병렬 실행"""
        import asyncio

        article = await self.get_article_by_id(article_id)
        if not article:
            return None

        # 병렬로 관련 데이터 조회
        comments_task = self._get_comments(article_id)
        likes_task = self._get_likes_count(article_id)

        comments, likes_count = await asyncio.gather(
            comments_task,
            likes_task
        )

        return {
            'article': article,
            'comments': comments,
            'likes_count': likes_count
        }

    @sync_to_async
    def _get_comments(self, article_id: int):
        """댓글 조회"""
        from apps.comments.models import Comment
        return list(Comment.objects.filter(article_id=article_id))

    @sync_to_async
    def _get_likes_count(self, article_id: int) -> int:
        """좋아요 수 조회"""
        from apps.likes.models import Like
        return Like.objects.filter(article_id=article_id).count()
```

## 베스트 프랙티스

### 1. Service 명명 규칙
```python
# ✅ 좋은 예
class ArticleService:
    def create_article(...)  # 명확한 동사 사용
    def publish_article(...)
    def delete_article(...)

# ❌ 나쁜 예
class ArticleService:
    def create(...)  # 무엇을 생성하는지 불명확
    def do_publish(...)  # 불필요한 do_ 접두사
```

### 2. Transaction 관리
```python
from django.db import transaction

class ArticleService:
    @transaction.atomic
    def create_article_with_tags(self, title, content, tag_ids):
        """여러 데이터베이스 작업은 하나의 트랜잭션으로"""
        article = Article.objects.create(title=title, content=content)
        article.tags.set(tag_ids)
        # 에러 발생 시 모두 롤백
        return article
```

### 3. 외부 의존성 주입
```python
# services/notification_service.py
from typing import Protocol

class EmailProvider(Protocol):
    def send_email(self, to: str, subject: str, body: str) -> None:
        ...

class NotificationService:
    def __init__(self, email_provider: EmailProvider):
        self.email_provider = email_provider

    def send_welcome_email(self, user):
        self.email_provider.send_email(
            to=user.email,
            subject="Welcome!",
            body="Welcome to our platform"
        )

# 테스트 시 Mock으로 교체 가능
```

## 주의사항

### ❌ 피해야 할 패턴

1. **Service에서 HTTP 관련 코드 사용**
```python
# ❌ 나쁜 예
class ArticleService:
    def create_article(self, request):  # request 객체 사용 금지
        data = request.POST
        ...

# ✅ 좋은 예
class ArticleService:
    def create_article(self, title: str, content: str, author: User):
        ...
```

2. **Service에서 Serializer 직접 사용**
```python
# ❌ 나쁜 예
class ArticleService:
    def create_article(self, data):
        serializer = ArticleSerializer(data=data)
        serializer.is_valid(raise_exception=True)
        return serializer.save()

# ✅ 좋은 예
class ArticleService:
    def create_article(self, title: str, content: str):
        return Article.objects.create(title=title, content=content)
```

## 체크리스트

- [ ] Type hints를 모든 메서드에 추가했는가?
- [ ] 비즈니스 로직이 Service에 캡슐화되어 있는가?
- [ ] Transaction 처리가 적절한가?
- [ ] HTTP 관련 코드(request, response)가 없는가?
- [ ] Serializer를 직접 사용하지 않는가?
- [ ] 메서드명이 명확한가?
- [ ] 적절한 예외 처리가 되어 있는가?
- [ ] 재사용 가능한 구조인가?
- [ ] 단위 테스트가 용이한 구조인가?
- [ ] Service 간 의존성이 명확한가?
