# Django Exceptions 가이드라인

## 개요
명확한 예외 처리는 디버깅을 쉽게 하고, 클라이언트에게 유용한 에러 메시지를 제공합니다.

## 기본 원칙

### 1. Custom Exception 정의
```python
# exceptions.py
from rest_framework.exceptions import APIException
from rest_framework import status

class BaseAppException(APIException):
    """앱의 모든 커스텀 예외의 기본 클래스"""
    status_code = status.HTTP_400_BAD_REQUEST
    default_detail = 'A server error occurred.'
    default_code = 'error'

class ArticleNotFound(BaseAppException):
    """게시글을 찾을 수 없을 때"""
    status_code = status.HTTP_404_NOT_FOUND
    default_detail = 'Article not found.'
    default_code = 'article_not_found'

class ArticlePermissionDenied(BaseAppException):
    """게시글 권한이 없을 때"""
    status_code = status.HTTP_403_FORBIDDEN
    default_detail = 'You do not have permission to perform this action.'
    default_code = 'article_permission_denied'

class ArticleAlreadyPublished(BaseAppException):
    """게시글이 이미 발행되었을 때"""
    status_code = status.HTTP_400_BAD_REQUEST
    default_detail = 'Article is already published.'
    default_code = 'article_already_published'

class InvalidArticleData(BaseAppException):
    """잘못된 게시글 데이터"""
    status_code = status.HTTP_400_BAD_REQUEST
    default_detail = 'Invalid article data provided.'
    default_code = 'invalid_article_data'
```

### 2. 도메인별 Exception 그룹화
```python
# exceptions.py

# ========== Article Exceptions ==========
class ArticleNotFound(BaseAppException):
    status_code = status.HTTP_404_NOT_FOUND
    default_detail = 'Article not found.'
    default_code = 'article_not_found'

class ArticlePermissionDenied(BaseAppException):
    status_code = status.HTTP_403_FORBIDDEN
    default_detail = 'You do not have permission to access this article.'
    default_code = 'article_permission_denied'

# ========== Order Exceptions ==========
class OrderNotFound(BaseAppException):
    status_code = status.HTTP_404_NOT_FOUND
    default_detail = 'Order not found.'
    default_code = 'order_not_found'

class InsufficientStockError(BaseAppException):
    status_code = status.HTTP_400_BAD_REQUEST
    default_detail = 'Insufficient stock available.'
    default_code = 'insufficient_stock'

class InvalidOrderError(BaseAppException):
    status_code = status.HTTP_400_BAD_REQUEST
    default_detail = 'Invalid order.'
    default_code = 'invalid_order'

class PaymentFailedError(BaseAppException):
    status_code = status.HTTP_402_PAYMENT_REQUIRED
    default_detail = 'Payment processing failed.'
    default_code = 'payment_failed'

# ========== User Exceptions ==========
class UserNotFound(BaseAppException):
    status_code = status.HTTP_404_NOT_FOUND
    default_detail = 'User not found.'
    default_code = 'user_not_found'

class DuplicateEmailError(BaseAppException):
    status_code = status.HTTP_409_CONFLICT
    default_detail = 'Email already exists.'
    default_code = 'duplicate_email'

class InvalidCredentialsError(BaseAppException):
    status_code = status.HTTP_401_UNAUTHORIZED
    default_detail = 'Invalid credentials.'
    default_code = 'invalid_credentials'
```

### 3. 세밀한 에러 메시지
```python
# exceptions.py
class ArticleNotFound(BaseAppException):
    def __init__(self, article_id: int = None, detail: str = None):
        if article_id:
            detail = f'Article with id {article_id} not found.'
        super().__init__(detail=detail)

class InsufficientStockError(BaseAppException):
    def __init__(self, product_name: str = None, available: int = None, requested: int = None):
        if product_name and available is not None and requested is not None:
            detail = (
                f'Insufficient stock for {product_name}. '
                f'Available: {available}, Requested: {requested}'
            )
        else:
            detail = self.default_detail
        super().__init__(detail=detail)

# 사용 예시
raise ArticleNotFound(article_id=123)
raise InsufficientStockError(product_name='iPhone', available=5, requested=10)
```

### 4. Custom Exception Handler
```python
# handlers.py
from typing import Any, Dict, Optional
from rest_framework.views import exception_handler
from rest_framework.response import Response
from django.core.exceptions import ValidationError as DjangoValidationError
import logging

logger = logging.getLogger(__name__)

def custom_exception_handler(exc: Exception, context: Dict[str, Any]) -> Optional[Response]:
    """커스텀 예외 핸들러"""
    # DRF 기본 핸들러 호출
    response = exception_handler(exc, context)

    if response is not None:
        # 표준화된 에러 응답 형식
        custom_response_data = {
            'error': {
                'code': getattr(exc, 'default_code', 'error'),
                'message': response.data.get('detail', str(exc)),
                'details': response.data if isinstance(response.data, dict) else None
            }
        }
        response.data = custom_response_data

        # 로깅
        logger.error(
            f"API Error: {exc.__class__.__name__} - {str(exc)}",
            extra={'context': context}
        )

    else:
        # Django ValidationError 처리
        if isinstance(exc, DjangoValidationError):
            response = Response(
                {
                    'error': {
                        'code': 'validation_error',
                        'message': 'Validation failed',
                        'details': exc.message_dict if hasattr(exc, 'message_dict') else exc.messages
                    }
                },
                status=400
            )

        # 예상하지 못한 에러
        else:
            logger.exception(
                f"Unexpected error: {exc.__class__.__name__}",
                extra={'context': context}
            )
            response = Response(
                {
                    'error': {
                        'code': 'internal_error',
                        'message': 'An unexpected error occurred.'
                    }
                },
                status=500
            )

    return response

# settings.py에 등록
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'apps.core.handlers.custom_exception_handler',
}
```

### 5. 에러 응답 형식 표준화
```python
# handlers.py
def format_error_response(
    code: str,
    message: str,
    details: Any = None,
    field_errors: Dict[str, list] = None
) -> Dict[str, Any]:
    """표준화된 에러 응답 형식"""
    error_response = {
        'error': {
            'code': code,
            'message': message,
        }
    }

    if details:
        error_response['error']['details'] = details

    if field_errors:
        error_response['error']['field_errors'] = field_errors

    return error_response

# 응답 예시:
# {
#     "error": {
#         "code": "validation_error",
#         "message": "Invalid input data",
#         "field_errors": {
#             "email": ["This field is required."],
#             "password": ["Password must be at least 8 characters."]
#         }
#     }
# }
```

### 6. Service에서 Exception 사용
```python
# services/article_service.py
from apps.articles.exceptions import (
    ArticleNotFound,
    ArticlePermissionDenied,
    ArticleAlreadyPublished
)

class ArticleService:
    def publish_article(self, article_id: int, user) -> Article:
        """게시글 발행"""
        try:
            article = Article.objects.get(id=article_id)
        except Article.DoesNotExist:
            raise ArticleNotFound(article_id=article_id)

        if article.author != user:
            raise ArticlePermissionDenied(
                detail=f"User {user.username} does not have permission to publish this article"
            )

        if article.is_published:
            raise ArticleAlreadyPublished()

        article.is_published = True
        article.save(update_fields=['is_published'])

        return article
```

### 7. View에서 Exception 처리
```python
# views/article_viewset.py
from rest_framework.exceptions import NotFound, PermissionDenied, ValidationError
from apps.articles.exceptions import (
    ArticleNotFound,
    ArticlePermissionDenied,
    ArticleAlreadyPublished
)

class ArticleViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        try:
            article = self.service.publish_article(
                article_id=pk,
                user=request.user
            )
            serializer = ArticleDetailSerializer(article, context={'request': request})
            return Response(serializer.data)

        except ArticleNotFound as e:
            raise NotFound(detail=str(e))
        except ArticlePermissionDenied as e:
            raise PermissionDenied(detail=str(e))
        except ArticleAlreadyPublished as e:
            raise ValidationError(detail=str(e))
```

### 8. 여러 에러를 한 번에 수집
```python
# exceptions.py
from typing import List, Dict, Any

class MultipleValidationError(BaseAppException):
    """여러 검증 에러를 한 번에 반환"""
    status_code = status.HTTP_400_BAD_REQUEST
    default_code = 'multiple_validation_errors'

    def __init__(self, errors: List[Dict[str, Any]]):
        self.detail = {
            'errors': errors
        }

# 사용 예시
errors = []
if not title:
    errors.append({'field': 'title', 'message': 'Title is required'})
if len(content) < 10:
    errors.append({'field': 'content', 'message': 'Content must be at least 10 characters'})

if errors:
    raise MultipleValidationError(errors)
```

### 9. Rate Limit Exception
```python
# exceptions.py
class RateLimitExceeded(BaseAppException):
    """Rate limit 초과"""
    status_code = status.HTTP_429_TOO_MANY_REQUESTS
    default_detail = 'Rate limit exceeded. Please try again later.'
    default_code = 'rate_limit_exceeded'

    def __init__(self, retry_after: int = None):
        detail = self.default_detail
        if retry_after:
            detail = f'{detail} Retry after {retry_after} seconds.'
        super().__init__(detail=detail)
```

### 10. 외부 서비스 에러
```python
# exceptions.py
class ExternalServiceError(BaseAppException):
    """외부 서비스 호출 실패"""
    status_code = status.HTTP_503_SERVICE_UNAVAILABLE
    default_detail = 'External service is temporarily unavailable.'
    default_code = 'external_service_error'

class PaymentGatewayError(ExternalServiceError):
    """결제 게이트웨이 에러"""
    default_detail = 'Payment gateway is temporarily unavailable.'
    default_code = 'payment_gateway_error'

class EmailServiceError(ExternalServiceError):
    """이메일 서비스 에러"""
    default_detail = 'Email service is temporarily unavailable.'
    default_code = 'email_service_error'
```

## 베스트 프랙티스

### 1. Exception 계층 구조
```python
# exceptions.py
class BaseAppException(APIException):
    """최상위 예외"""
    pass

class ResourceNotFound(BaseAppException):
    """리소스 미발견 (404)"""
    status_code = status.HTTP_404_NOT_FOUND

class PermissionDenied(BaseAppException):
    """권한 거부 (403)"""
    status_code = status.HTTP_403_FORBIDDEN

class ValidationError(BaseAppException):
    """유효성 검증 실패 (400)"""
    status_code = status.HTTP_400_BAD_REQUEST

# 구체적인 예외들
class ArticleNotFound(ResourceNotFound):
    default_code = 'article_not_found'

class CommentNotFound(ResourceNotFound):
    default_code = 'comment_not_found'
```

### 2. HTTP 상태 코드 활용
```python
# 200 범위 - 성공
HTTP_200_OK = 200
HTTP_201_CREATED = 201
HTTP_204_NO_CONTENT = 204

# 400 범위 - 클라이언트 에러
HTTP_400_BAD_REQUEST = 400          # 잘못된 요청
HTTP_401_UNAUTHORIZED = 401         # 인증 필요
HTTP_403_FORBIDDEN = 403            # 권한 없음
HTTP_404_NOT_FOUND = 404            # 리소스 없음
HTTP_409_CONFLICT = 409             # 충돌 (중복 등)
HTTP_422_UNPROCESSABLE_ENTITY = 422 # 유효성 검증 실패
HTTP_429_TOO_MANY_REQUESTS = 429    # Rate limit 초과

# 500 범위 - 서버 에러
HTTP_500_INTERNAL_SERVER_ERROR = 500  # 내부 서버 에러
HTTP_503_SERVICE_UNAVAILABLE = 503    # 서비스 이용 불가
```

### 3. 로깅
```python
# services/article_service.py
import logging

logger = logging.getLogger(__name__)

class ArticleService:
    def delete_article(self, article_id: int, user):
        try:
            article = Article.objects.get(id=article_id)
        except Article.DoesNotExist:
            logger.warning(f"Attempted to delete non-existent article: {article_id}")
            raise ArticleNotFound(article_id=article_id)

        if article.author != user and not user.is_staff:
            logger.warning(
                f"User {user.id} attempted to delete article {article_id} without permission"
            )
            raise ArticlePermissionDenied()

        logger.info(f"Article {article_id} deleted by user {user.id}")
        article.delete()
```

## 주의사항

### ❌ 피해야 할 패턴

1. **일반적인 Exception 사용**
```python
# ❌ 나쁜 예
raise Exception("Article not found")

# ✅ 좋은 예
raise ArticleNotFound(article_id=123)
```

2. **불명확한 에러 메시지**
```python
# ❌ 나쁜 예
raise ValidationError("Invalid data")

# ✅ 좋은 예
raise ValidationError("Title must be at least 5 characters long")
```

3. **에러 무시**
```python
# ❌ 나쁜 예
try:
    article = Article.objects.get(id=article_id)
except Article.DoesNotExist:
    pass  # 에러 무시

# ✅ 좋은 예
try:
    article = Article.objects.get(id=article_id)
except Article.DoesNotExist:
    raise ArticleNotFound(article_id=article_id)
```

4. **너무 광범위한 Exception 처리**
```python
# ❌ 나쁜 예
try:
    # 복잡한 로직
    ...
except Exception as e:
    return Response({'error': 'Something went wrong'})

# ✅ 좋은 예
try:
    # 복잡한 로직
    ...
except ArticleNotFound as e:
    raise NotFound(str(e))
except ArticlePermissionDenied as e:
    raise PermissionDenied(str(e))
except ValidationError as e:
    raise
```

## 체크리스트

- [ ] 명확한 Exception 클래스를 정의했는가?
- [ ] 적절한 HTTP 상태 코드를 사용하는가?
- [ ] 유용한 에러 메시지를 제공하는가?
- [ ] Exception 계층 구조가 논리적인가?
- [ ] 에러를 적절히 로깅하고 있는가?
- [ ] Custom exception handler를 구현했는가?
- [ ] 표준화된 에러 응답 형식을 사용하는가?
- [ ] 도메인별로 Exception을 그룹화했는가?
- [ ] 에러를 무시하지 않고 처리하는가?
- [ ] 민감한 정보를 에러 메시지에 포함하지 않는가?
