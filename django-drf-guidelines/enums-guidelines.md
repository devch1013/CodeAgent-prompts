# Django Enums 가이드라인

## 개요
Enum을 사용하면 코드의 가독성과 유지보수성이 향상되고, 매직 스트링/넘버를 방지할 수 있습니다.

## 기본 원칙

### 1. Python Enum 활용 (Python 3.11+)
```python
# enums.py
from enum import Enum, IntEnum
from typing import List, Tuple

class OrderStatus(str, Enum):
    """주문 상태"""
    PENDING = 'pending'
    PAID = 'paid'
    PROCESSING = 'processing'
    SHIPPED = 'shipped'
    DELIVERED = 'delivered'
    CANCELLED = 'cancelled'
    REFUNDED = 'refunded'

    @classmethod
    def choices(cls) -> List[Tuple[str, str]]:
        """Django choices 형식 반환"""
        return [(item.value, item.name.replace('_', ' ').title()) for item in cls]

    @classmethod
    def values(cls) -> List[str]:
        """모든 값 반환"""
        return [item.value for item in cls]

    @classmethod
    def names(cls) -> List[str]:
        """모든 이름 반환"""
        return [item.name for item in cls]

# 사용 예시
# OrderStatus.PENDING.value  # 'pending'
# OrderStatus.choices()  # [('pending', 'Pending'), ('paid', 'Paid'), ...]
```

### 2. Django 3.0+ TextChoices/IntegerChoices
```python
# enums.py
from django.db import models

class OrderStatus(models.TextChoices):
    """주문 상태 (TextChoices)"""
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'
    PROCESSING = 'processing', 'Processing'
    SHIPPED = 'shipped', 'Shipped'
    DELIVERED = 'delivered', 'Delivered'
    CANCELLED = 'cancelled', 'Cancelled'
    REFUNDED = 'refunded', 'Refunded'

class Priority(models.IntegerChoices):
    """우선순위 (IntegerChoices)"""
    LOW = 1, 'Low'
    MEDIUM = 2, 'Medium'
    HIGH = 3, 'High'
    URGENT = 4, 'Urgent'

class ArticleStatus(models.TextChoices):
    """게시글 상태"""
    DRAFT = 'draft', 'Draft'
    PUBLISHED = 'published', 'Published'
    ARCHIVED = 'archived', 'Archived'

# 모델에서 사용
class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )

class Task(models.Model):
    priority = models.IntegerField(
        choices=Priority.choices,
        default=Priority.MEDIUM
    )
```

### 3. 비즈니스 로직을 가진 Enum
```python
# enums.py
from enum import Enum
from typing import List

class OrderStatus(str, Enum):
    PENDING = 'pending'
    PAID = 'paid'
    PROCESSING = 'processing'
    SHIPPED = 'shipped'
    DELIVERED = 'delivered'
    CANCELLED = 'cancelled'
    REFUNDED = 'refunded'

    def can_cancel(self) -> bool:
        """취소 가능한 상태인지 확인"""
        return self in [
            OrderStatus.PENDING,
            OrderStatus.PAID,
            OrderStatus.PROCESSING
        ]

    def can_refund(self) -> bool:
        """환불 가능한 상태인지 확인"""
        return self in [
            OrderStatus.PAID,
            OrderStatus.PROCESSING,
            OrderStatus.SHIPPED,
            OrderStatus.DELIVERED
        ]

    def is_completed(self) -> bool:
        """완료된 상태인지 확인"""
        return self in [
            OrderStatus.DELIVERED,
            OrderStatus.CANCELLED,
            OrderStatus.REFUNDED
        ]

    def next_statuses(self) -> List['OrderStatus']:
        """다음 가능한 상태들"""
        transitions = {
            OrderStatus.PENDING: [OrderStatus.PAID, OrderStatus.CANCELLED],
            OrderStatus.PAID: [OrderStatus.PROCESSING, OrderStatus.REFUNDED],
            OrderStatus.PROCESSING: [OrderStatus.SHIPPED, OrderStatus.CANCELLED],
            OrderStatus.SHIPPED: [OrderStatus.DELIVERED],
            OrderStatus.DELIVERED: [OrderStatus.REFUNDED],
            OrderStatus.CANCELLED: [],
            OrderStatus.REFUNDED: []
        }
        return transitions.get(self, [])

    @classmethod
    def choices(cls):
        return [(item.value, item.name.replace('_', ' ').title()) for item in cls]

# 사용 예시
status = OrderStatus.PAID
if status.can_cancel():
    # 취소 가능
    pass

next_statuses = status.next_statuses()  # [OrderStatus.PROCESSING, OrderStatus.REFUNDED]
```

### 4. 권한 관련 Enum
```python
# enums.py
from enum import IntFlag

class Permission(IntFlag):
    """비트 플래그를 사용한 권한"""
    NONE = 0
    READ = 1
    WRITE = 2
    DELETE = 4
    ADMIN = 8

    READ_WRITE = READ | WRITE
    ALL = READ | WRITE | DELETE | ADMIN

# 사용 예시
user_permissions = Permission.READ | Permission.WRITE

if user_permissions & Permission.READ:
    # 읽기 권한 있음
    pass

if user_permissions & Permission.WRITE:
    # 쓰기 권한 있음
    pass

# 또는 간단한 버전
class UserRole(models.TextChoices):
    """사용자 역할"""
    GUEST = 'guest', 'Guest'
    USER = 'user', 'User'
    MODERATOR = 'moderator', 'Moderator'
    ADMIN = 'admin', 'Admin'
    SUPERUSER = 'superuser', 'Superuser'
```

### 5. 카테고리/타입 Enum
```python
# enums.py
class ArticleCategory(models.TextChoices):
    """게시글 카테고리"""
    TECHNOLOGY = 'technology', 'Technology'
    SCIENCE = 'science', 'Science'
    BUSINESS = 'business', 'Business'
    HEALTH = 'health', 'Health'
    ENTERTAINMENT = 'entertainment', 'Entertainment'

class PaymentMethod(models.TextChoices):
    """결제 수단"""
    CREDIT_CARD = 'credit_card', 'Credit Card'
    DEBIT_CARD = 'debit_card', 'Debit Card'
    PAYPAL = 'paypal', 'PayPal'
    BANK_TRANSFER = 'bank_transfer', 'Bank Transfer'
    CASH = 'cash', 'Cash'

class NotificationType(models.TextChoices):
    """알림 타입"""
    EMAIL = 'email', 'Email'
    SMS = 'sms', 'SMS'
    PUSH = 'push', 'Push Notification'
    IN_APP = 'in_app', 'In-App'
```

### 6. 에러 코드 Enum
```python
# enums.py
class ErrorCode(str, Enum):
    """API 에러 코드"""
    # 인증 관련 (1000번대)
    INVALID_CREDENTIALS = '1001'
    TOKEN_EXPIRED = '1002'
    TOKEN_INVALID = '1003'
    UNAUTHORIZED = '1004'

    # 권한 관련 (2000번대)
    PERMISSION_DENIED = '2001'
    FORBIDDEN = '2002'

    # 리소스 관련 (3000번대)
    RESOURCE_NOT_FOUND = '3001'
    RESOURCE_ALREADY_EXISTS = '3002'

    # 유효성 검증 (4000번대)
    VALIDATION_ERROR = '4001'
    INVALID_INPUT = '4002'
    MISSING_REQUIRED_FIELD = '4003'

    # 비즈니스 로직 (5000번대)
    INSUFFICIENT_STOCK = '5001'
    PAYMENT_FAILED = '5002'
    ORDER_ALREADY_CANCELLED = '5003'

    def get_message(self) -> str:
        """에러 메시지 반환"""
        messages = {
            ErrorCode.INVALID_CREDENTIALS: 'Invalid username or password',
            ErrorCode.TOKEN_EXPIRED: 'Authentication token has expired',
            ErrorCode.TOKEN_INVALID: 'Invalid authentication token',
            ErrorCode.UNAUTHORIZED: 'Authentication required',
            ErrorCode.PERMISSION_DENIED: 'You do not have permission to perform this action',
            ErrorCode.RESOURCE_NOT_FOUND: 'Requested resource not found',
            ErrorCode.INSUFFICIENT_STOCK: 'Insufficient stock available',
        }
        return messages.get(self, 'An error occurred')
```

### 7. 상태 머신 Enum
```python
# enums.py
from typing import Dict, Set

class TaskStatus(str, Enum):
    """작업 상태 (상태 머신)"""
    TODO = 'todo'
    IN_PROGRESS = 'in_progress'
    IN_REVIEW = 'in_review'
    DONE = 'done'
    CANCELLED = 'cancelled'

    @classmethod
    def get_transitions(cls) -> Dict[str, Set[str]]:
        """상태 전환 규칙"""
        return {
            cls.TODO.value: {cls.IN_PROGRESS.value, cls.CANCELLED.value},
            cls.IN_PROGRESS.value: {cls.IN_REVIEW.value, cls.TODO.value, cls.CANCELLED.value},
            cls.IN_REVIEW.value: {cls.DONE.value, cls.IN_PROGRESS.value},
            cls.DONE.value: set(),
            cls.CANCELLED.value: {cls.TODO.value}
        }

    def can_transition_to(self, target_status: 'TaskStatus') -> bool:
        """특정 상태로 전환 가능한지 확인"""
        transitions = self.get_transitions()
        return target_status.value in transitions.get(self.value, set())

# 사용 예시
current_status = TaskStatus.IN_PROGRESS
if current_status.can_transition_to(TaskStatus.DONE):
    # 상태 전환 불가
    raise ValidationError("Cannot transition from IN_PROGRESS to DONE directly")
```

### 8. 다국어 지원 Enum
```python
# enums.py
from django.utils.translation import gettext_lazy as _

class OrderStatus(models.TextChoices):
    """주문 상태 (다국어 지원)"""
    PENDING = 'pending', _('Pending')
    PAID = 'paid', _('Paid')
    PROCESSING = 'processing', _('Processing')
    SHIPPED = 'shipped', _('Shipped')
    DELIVERED = 'delivered', _('Delivered')
    CANCELLED = 'cancelled', _('Cancelled')
    REFUNDED = 'refunded', _('Refunded')
```

### 9. Enum과 Model 통합
```python
# models.py
from apps.orders.enums import OrderStatus

class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )

    def can_be_cancelled(self) -> bool:
        """취소 가능한지 확인"""
        status_enum = OrderStatus(self.status)
        return status_enum.can_cancel()

    def cancel(self) -> None:
        """주문 취소"""
        status_enum = OrderStatus(self.status)
        if not status_enum.can_cancel():
            raise ValidationError(f"Cannot cancel order with status {self.status}")

        self.status = OrderStatus.CANCELLED.value
        self.save(update_fields=['status'])
```

### 10. DRF Serializer와 Enum
```python
# serializers.py
from rest_framework import serializers
from apps.orders.enums import OrderStatus
from apps.orders.models import Order

class OrderSerializer(serializers.ModelSerializer):
    status = serializers.ChoiceField(choices=OrderStatus.choices)
    status_display = serializers.CharField(source='get_status_display', read_only=True)
    can_cancel = serializers.SerializerMethodField()

    class Meta:
        model = Order
        fields = ['id', 'status', 'status_display', 'can_cancel']

    def get_can_cancel(self, obj: Order) -> bool:
        status_enum = OrderStatus(obj.status)
        return status_enum.can_cancel()

# 응답 예시:
# {
#     "id": 1,
#     "status": "paid",
#     "status_display": "Paid",
#     "can_cancel": true
# }
```

### 11. Filtering과 Enum
```python
# views/order_viewset.py
from django_filters import rest_framework as filters
from apps.orders.enums import OrderStatus

class OrderFilter(filters.FilterSet):
    """주문 필터"""
    status = filters.MultipleChoiceFilter(choices=OrderStatus.choices)
    status_in = filters.MultipleChoiceFilter(
        field_name='status',
        lookup_expr='in',
        choices=OrderStatus.choices
    )

    class Meta:
        model = Order
        fields = ['status']

# 사용 예시: /api/orders/?status=pending&status=paid
```

### 12. Enum 유효성 검증
```python
# validators.py
from django.core.exceptions import ValidationError
from apps.orders.enums import OrderStatus

def validate_order_status(value: str) -> None:
    """주문 상태 유효성 검증"""
    if value not in OrderStatus.values():
        raise ValidationError(
            f'{value} is not a valid order status. '
            f'Valid choices are: {", ".join(OrderStatus.values())}'
        )

# models.py
from apps.core.validators import validate_order_status

class Order(models.Model):
    status = models.CharField(
        max_length=20,
        validators=[validate_order_status]
    )
```

## 베스트 프랙티스

### 1. 명확한 Enum 이름
```python
# ✅ 좋은 예
class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'

class PaymentMethod(models.TextChoices):
    CREDIT_CARD = 'credit_card', 'Credit Card'
    PAYPAL = 'paypal', 'PayPal'

# ❌ 나쁜 예
class Status(models.TextChoices):  # 너무 일반적
    S1 = '1', 'Status 1'  # 의미 불명확
    S2 = '2', 'Status 2'
```

### 2. Enum 값 형식
```python
# ✅ 좋은 예 - 소문자 스네이크 케이스
class OrderStatus(str, Enum):
    PENDING = 'pending'
    IN_PROGRESS = 'in_progress'
    COMPLETED = 'completed'

# ❌ 나쁜 예 - 일관성 없음
class OrderStatus(str, Enum):
    PENDING = 'Pending'
    InProgress = 'in-progress'
    completed = 'COMPLETED'
```

### 3. Enum 그룹화
```python
# enums.py
# 도메인별로 Enum 그룹화

# ========== Order Enums ==========
class OrderStatus(models.TextChoices):
    """주문 상태"""
    pass

class PaymentMethod(models.TextChoices):
    """결제 수단"""
    pass

# ========== Article Enums ==========
class ArticleStatus(models.TextChoices):
    """게시글 상태"""
    pass

class ArticleCategory(models.TextChoices):
    """게시글 카테고리"""
    pass

# ========== User Enums ==========
class UserRole(models.TextChoices):
    """사용자 역할"""
    pass

class AccountStatus(models.TextChoices):
    """계정 상태"""
    pass
```

## 주의사항

### ❌ 피해야 할 패턴

1. **매직 스트링 사용**
```python
# ❌ 나쁜 예
if order.status == 'pending':
    pass

# ✅ 좋은 예
if order.status == OrderStatus.PENDING.value:
    pass
```

2. **Enum 없이 하드코딩**
```python
# ❌ 나쁜 예
class Order(models.Model):
    status = models.CharField(max_length=20)

# ✅ 좋은 예
class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )
```

3. **불필요하게 복잡한 Enum**
```python
# ❌ 나쁜 예 - 너무 많은 로직
class OrderStatus(str, Enum):
    PENDING = 'pending'

    def send_email(self):  # Enum에 너무 많은 책임
        pass

    def update_inventory(self):
        pass

# ✅ 좋은 예 - 간단한 유틸리티만
class OrderStatus(str, Enum):
    PENDING = 'pending'

    def is_completed(self) -> bool:
        return self in [OrderStatus.DELIVERED, OrderStatus.CANCELLED]
```

## 체크리스트

- [ ] 매직 스트링/넘버를 Enum으로 대체했는가?
- [ ] Django TextChoices/IntegerChoices를 사용하는가?
- [ ] Enum 이름이 명확하고 일관성 있는가?
- [ ] choices() 메서드를 제공하는가?
- [ ] 비즈니스 로직이 적절히 캡슐화되어 있는가?
- [ ] 상태 전환 규칙이 명확한가?
- [ ] 다국어 지원을 고려했는가?
- [ ] 도메인별로 Enum을 그룹화했는가?
- [ ] Type hints를 사용하는가?
- [ ] Serializer에서 Enum을 적절히 활용하는가?
