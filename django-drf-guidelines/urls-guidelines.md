# Django REST Framework URLs 가이드라인

## 개요
RESTful API의 URL 구조를 명확하고 일관성 있게 설계하는 가이드라인입니다.

## 기본 원칙

### 1. ViewSet과 Router 활용 (권장)
```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from apps.articles.views import ArticleViewSet, CommentViewSet

# Router 설정
router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')
router.register(r'comments', CommentViewSet, basename='comment')

urlpatterns = [
    path('api/', include(router.urls)),
]

# 생성되는 URL 패턴:
# GET    /api/articles/              -> list
# POST   /api/articles/              -> create
# GET    /api/articles/{pk}/         -> retrieve
# PUT    /api/articles/{pk}/         -> update
# PATCH  /api/articles/{pk}/         -> partial_update
# DELETE /api/articles/{pk}/         -> destroy
```

### 2. Nested Routes
```python
# urls.py
from rest_framework_nested import routers

# 기본 router
router = routers.DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')

# Nested router (게시글의 댓글)
articles_router = routers.NestedDefaultRouter(router, r'articles', lookup='article')
articles_router.register(r'comments', CommentViewSet, basename='article-comments')

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/', include(articles_router.urls)),
]

# 생성되는 URL 패턴:
# GET    /api/articles/{article_pk}/comments/              -> 특정 게시글의 댓글 목록
# POST   /api/articles/{article_pk}/comments/              -> 특정 게시글에 댓글 작성
# GET    /api/articles/{article_pk}/comments/{pk}/         -> 댓글 상세
# PUT    /api/articles/{article_pk}/comments/{pk}/         -> 댓글 수정
# DELETE /api/articles/{article_pk}/comments/{pk}/         -> 댓글 삭제
```

### 3. Custom Actions
```python
# views/article_viewset.py
from rest_framework.decorators import action
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """게시글 발행"""
        # POST /api/articles/{pk}/publish/
        pass

    @action(detail=True, methods=['post', 'delete'])
    def like(self, request, pk=None):
        """게시글 좋아요/좋아요 취소"""
        # POST   /api/articles/{pk}/like/
        # DELETE /api/articles/{pk}/like/
        pass

    @action(detail=False, methods=['get'])
    def trending(self, request):
        """인기 게시글"""
        # GET /api/articles/trending/
        pass

    @action(detail=False, methods=['get'])
    def my_articles(self, request):
        """내 게시글"""
        # GET /api/articles/my_articles/
        pass
```

### 4. API Versioning
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}

# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter

# v1 router
router_v1 = DefaultRouter()
router_v1.register(r'articles', ArticleViewSetV1, basename='article')
router_v1.register(r'users', UserViewSetV1, basename='user')

# v2 router
router_v2 = DefaultRouter()
router_v2.register(r'articles', ArticleViewSetV2, basename='article')
router_v2.register(r'users', UserViewSetV2, basename='user')

urlpatterns = [
    path('api/v1/', include(router_v1.urls)),
    path('api/v2/', include(router_v2.urls)),
]

# URL 패턴:
# /api/v1/articles/
# /api/v2/articles/
```

### 5. 앱별 URL 구조화
```python
# project/urls.py (메인)
from django.urls import path, include

urlpatterns = [
    path('api/v1/articles/', include('apps.articles.urls')),
    path('api/v1/users/', include('apps.users.urls')),
    path('api/v1/orders/', include('apps.orders.urls')),
]

# apps/articles/urls.py
from rest_framework.routers import DefaultRouter
from apps.articles.views import ArticleViewSet, CommentViewSet

router = DefaultRouter()
router.register(r'', ArticleViewSet, basename='article')
router.register(r'comments', CommentViewSet, basename='comment')

urlpatterns = router.urls

# apps/users/urls.py
from rest_framework.routers import DefaultRouter
from apps.users.views import UserViewSet

router = DefaultRouter()
router.register(r'', UserViewSet, basename='user')

urlpatterns = router.urls

# 결과 URL:
# /api/v1/articles/
# /api/v1/articles/comments/
# /api/v1/users/
```

### 6. 인증 관련 URL
```python
# apps/authentication/urls.py
from django.urls import path
from apps.authentication.views import (
    LoginView,
    LogoutView,
    RegisterView,
    RefreshTokenView,
    PasswordResetView,
    PasswordResetConfirmView,
)

urlpatterns = [
    path('login/', LoginView.as_view(), name='login'),
    path('logout/', LogoutView.as_view(), name='logout'),
    path('register/', RegisterView.as_view(), name='register'),
    path('token/refresh/', RefreshTokenView.as_view(), name='token_refresh'),
    path('password/reset/', PasswordResetView.as_view(), name='password_reset'),
    path('password/reset/confirm/', PasswordResetConfirmView.as_view(), name='password_reset_confirm'),
]

# project/urls.py
urlpatterns = [
    path('api/v1/auth/', include('apps.authentication.urls')),
]

# 결과 URL:
# POST /api/v1/auth/login/
# POST /api/v1/auth/logout/
# POST /api/v1/auth/register/
# POST /api/v1/auth/token/refresh/
# POST /api/v1/auth/password/reset/
# POST /api/v1/auth/password/reset/confirm/
```

### 7. 검색 및 필터링 URL
```python
# views/article_viewset.py
class ArticleViewSet(viewsets.ModelViewSet):
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['category', 'author', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'updated_at', 'title']

# URL 사용 예시:
# GET /api/articles/?category=tech
# GET /api/articles/?author=1&is_published=true
# GET /api/articles/?search=django
# GET /api/articles/?ordering=-created_at
# GET /api/articles/?category=tech&search=django&ordering=-created_at
```

### 8. 페이지네이션 URL
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20
}

# URL 사용 예시:
# GET /api/articles/?page=1
# GET /api/articles/?page=2
# GET /api/articles/?page=1&page_size=50

# 응답 형식:
# {
#     "count": 100,
#     "next": "http://api.example.com/api/articles/?page=2",
#     "previous": null,
#     "results": [...]
# }
```

### 9. Bulk Operations
```python
# views/article_viewset.py
from rest_framework.decorators import action

class ArticleViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=['post'])
    def bulk_create(self, request):
        """여러 게시글 한번에 생성"""
        # POST /api/articles/bulk_create/
        # Body: [{"title": "...", "content": "..."}, {...}]
        pass

    @action(detail=False, methods=['patch'])
    def bulk_update(self, request):
        """여러 게시글 한번에 수정"""
        # PATCH /api/articles/bulk_update/
        # Body: [{"id": 1, "title": "..."}, {"id": 2, "title": "..."}]
        pass

    @action(detail=False, methods=['delete'])
    def bulk_delete(self, request):
        """여러 게시글 한번에 삭제"""
        # DELETE /api/articles/bulk_delete/?ids=1,2,3
        pass
```

### 10. Export/Import URL
```python
# views/article_viewset.py
class ArticleViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=['get'])
    def export(self, request):
        """게시글 내보내기"""
        # GET /api/articles/export/?format=csv
        # GET /api/articles/export/?format=json
        # GET /api/articles/export/?format=xlsx
        pass

    @action(detail=True, methods=['get'])
    def export_single(self, request, pk=None):
        """특정 게시글 내보내기"""
        # GET /api/articles/{pk}/export/?format=pdf
        pass

    @action(detail=False, methods=['post'])
    def import_data(self, request):
        """게시글 가져오기"""
        # POST /api/articles/import/
        # Body: file upload
        pass
```

### 11. 통계 및 분석 URL
```python
# views/analytics_viewset.py
from rest_framework.views import APIView

class ArticleAnalyticsView(APIView):
    """게시글 통계"""
    def get(self, request):
        # GET /api/analytics/articles/
        pass

class UserAnalyticsView(APIView):
    """사용자 통계"""
    def get(self, request):
        # GET /api/analytics/users/
        pass

# urls.py
urlpatterns = [
    path('api/analytics/articles/', ArticleAnalyticsView.as_view()),
    path('api/analytics/users/', UserAnalyticsView.as_view()),
]
```

### 12. Health Check & Status
```python
# views/health_views.py
from rest_framework.views import APIView
from rest_framework.response import Response

class HealthCheckView(APIView):
    """서비스 상태 확인"""
    permission_classes = []

    def get(self, request):
        # GET /api/health/
        return Response({'status': 'ok'})

class StatusView(APIView):
    """서비스 상세 상태"""
    def get(self, request):
        # GET /api/status/
        return Response({
            'status': 'ok',
            'version': '1.0.0',
            'database': 'connected',
            'redis': 'connected'
        })

# urls.py
urlpatterns = [
    path('api/health/', HealthCheckView.as_view()),
    path('api/status/', StatusView.as_view()),
]
```

## RESTful URL 설계 원칙

### 1. 리소스 명명 규칙
```python
# ✅ 좋은 예 - 복수형 명사 사용
/api/articles/
/api/users/
/api/orders/
/api/products/

# ❌ 나쁜 예 - 동사 사용
/api/getArticles/
/api/createUser/
/api/deleteOrder/
```

### 2. 계층 구조 표현
```python
# ✅ 좋은 예
/api/articles/{article_id}/comments/
/api/users/{user_id}/orders/
/api/orders/{order_id}/items/

# ❌ 나쁜 예 - 너무 깊은 중첩
/api/users/{user_id}/articles/{article_id}/comments/{comment_id}/replies/
# 대신 쿼리 파라미터 사용:
/api/comments/?article_id={article_id}
```

### 3. HTTP 메서드 활용
```python
# 게시글 CRUD
GET    /api/articles/           # 목록 조회
POST   /api/articles/           # 생성
GET    /api/articles/{id}/      # 상세 조회
PUT    /api/articles/{id}/      # 전체 수정
PATCH  /api/articles/{id}/      # 부분 수정
DELETE /api/articles/{id}/      # 삭제

# 커스텀 액션
POST   /api/articles/{id}/publish/      # 발행
POST   /api/articles/{id}/like/         # 좋아요
DELETE /api/articles/{id}/like/         # 좋아요 취소
GET    /api/articles/trending/          # 인기 게시글
```

### 4. 쿼리 파라미터 활용
```python
# 필터링
GET /api/articles/?category=tech
GET /api/articles/?author=1&status=published

# 검색
GET /api/articles/?search=django

# 정렬
GET /api/articles/?ordering=-created_at
GET /api/articles/?ordering=title

# 페이지네이션
GET /api/articles/?page=2&page_size=20

# 필드 선택 (sparse fieldsets)
GET /api/articles/?fields=id,title,author

# 관계 포함 (include)
GET /api/articles/?include=author,comments
```

## URL 네이밍 규칙

### 1. URL 케이스
```python
# ✅ 좋은 예 - kebab-case 또는 snake_case
/api/articles/
/api/user-profiles/
/api/order-items/

# ❌ 나쁜 예 - camelCase
/api/userProfiles/
/api/orderItems/
```

### 2. 버전 관리
```python
# 방법 1: URL Path (권장)
/api/v1/articles/
/api/v2/articles/

# 방법 2: Query Parameter
/api/articles/?version=v1

# 방법 3: Header
# Accept: application/vnd.myapp.v1+json
```

## 보안 고려사항

### 1. 민감한 정보 노출 방지
```python
# ❌ 나쁜 예
/api/users/?password=xxx
/api/orders/?credit_card=1234

# ✅ 좋은 예
POST /api/auth/login/
# Body: {"password": "xxx"}
```

### 2. Rate Limiting
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day'
    }
}
```

## 완전한 URL 구조 예시

```python
# project/urls.py
from django.urls import path, include

urlpatterns = [
    # Health Check
    path('health/', include('apps.core.health_urls')),

    # Authentication
    path('api/v1/auth/', include('apps.authentication.urls')),

    # Main APIs
    path('api/v1/users/', include('apps.users.urls')),
    path('api/v1/articles/', include('apps.articles.urls')),
    path('api/v1/orders/', include('apps.orders.urls')),

    # Analytics
    path('api/v1/analytics/', include('apps.analytics.urls')),
]

# 결과 URL 구조:
"""
/health/

/api/v1/auth/login/
/api/v1/auth/logout/
/api/v1/auth/register/
/api/v1/auth/token/refresh/

/api/v1/users/
/api/v1/users/{id}/
/api/v1/users/{id}/articles/
/api/v1/users/{id}/orders/

/api/v1/articles/
/api/v1/articles/{id}/
/api/v1/articles/{id}/publish/
/api/v1/articles/{id}/like/
/api/v1/articles/{id}/comments/
/api/v1/articles/trending/
/api/v1/articles/my_articles/

/api/v1/orders/
/api/v1/orders/{id}/
/api/v1/orders/{id}/cancel/
/api/v1/orders/{id}/items/

/api/v1/analytics/articles/
/api/v1/analytics/users/
"""
```

## 체크리스트

- [ ] ViewSet과 Router를 사용하는가?
- [ ] RESTful 원칙을 따르는가?
- [ ] URL이 명확하고 일관성 있는가?
- [ ] API 버저닝을 적용했는가?
- [ ] 적절한 HTTP 메서드를 사용하는가?
- [ ] 쿼리 파라미터를 활용하는가?
- [ ] Nested routes가 적절한가?
- [ ] 보안을 고려했는가?
- [ ] 문서화가 되어 있는가?
- [ ] URL 구조가 직관적인가?
