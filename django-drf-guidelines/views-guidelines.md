# Django REST Framework Views 가이드라인

## 개요
DRF Views는 HTTP 요청을 받아 적절한 응답을 반환하는 역할만 담당해야 합니다. 비즈니스 로직은 Service Layer로 분리합니다.

## Views의 역할

1. **HTTP 요청 처리**
   - 요청 데이터 파싱
   - 인증/권한 검증
   - 입력 유효성 검증 (Serializer 활용)

2. **Service 호출**
   - 비즈니스 로직은 Service에 위임

3. **HTTP 응답 반환**
   - 적절한 HTTP 상태 코드와 함께 응답

## 기본 구조

### 1. APIView 기본 패턴
```python
# views/article_views.py
from typing import Any
from rest_framework import status
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

from apps.articles.serializers import (
    ArticleListSerializer,
    ArticleDetailSerializer,
    ArticleCreateSerializer,
    ArticleUpdateSerializer
)
from apps.articles.services.article_service import ArticleService
from apps.articles.exceptions import ArticleNotFound, ArticlePermissionDenied

class ArticleListCreateAPIView(APIView):
    """게시글 목록 조회 및 생성"""
    permission_classes = [IsAuthenticated]

    def __init__(self, **kwargs: Any):
        super().__init__(**kwargs)
        self.service = ArticleService()

    def get(self, request: Request) -> Response:
        """게시글 목록 조회"""
        # Service 호출
        articles = self.service.get_published_articles()

        # Serialization
        serializer = ArticleListSerializer(articles, many=True, context={'request': request})

        return Response(serializer.data, status=status.HTTP_200_OK)

    def post(self, request: Request) -> Response:
        """게시글 생성"""
        # 입력 검증
        serializer = ArticleCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Service 호출
        article = self.service.create_article(
            title=serializer.validated_data['title'],
            content=serializer.validated_data['content'],
            author=request.user,
            tag_ids=serializer.validated_data.get('tag_ids', [])
        )

        # 응답
        response_serializer = ArticleDetailSerializer(article, context={'request': request})
        return Response(response_serializer.data, status=status.HTTP_201_CREATED)
```

### 2. ViewSet 패턴 (권장)
```python
# views/article_viewset.py
from typing import Any
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, AllowAny

from apps.articles.models import Article
from apps.articles.serializers import (
    ArticleListSerializer,
    ArticleDetailSerializer,
    ArticleCreateSerializer,
    ArticleUpdateSerializer
)
from apps.articles.services.article_service import ArticleService
from apps.articles.permissions import IsAuthorOrReadOnly

class ArticleViewSet(viewsets.ModelViewSet):
    """게시글 ViewSet"""
    queryset = Article.objects.all()
    service_class = ArticleService

    def __init__(self, **kwargs: Any):
        super().__init__(**kwargs)
        self.service = self.service_class()

    def get_permissions(self):
        """액션별 권한 설정"""
        if self.action in ['list', 'retrieve']:
            return [AllowAny()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [IsAuthenticated(), IsAuthorOrReadOnly()]
        return [IsAuthenticated()]

    def get_serializer_class(self):
        """액션별 Serializer 선택"""
        if self.action == 'list':
            return ArticleListSerializer
        elif self.action == 'create':
            return ArticleCreateSerializer
        elif self.action in ['update', 'partial_update']:
            return ArticleUpdateSerializer
        return ArticleDetailSerializer

    def get_queryset(self):
        """쿼리셋 최적화"""
        queryset = super().get_queryset()

        if self.action == 'list':
            return queryset.select_related('author').prefetch_related('tags')
        elif self.action == 'retrieve':
            return queryset.select_related('author', 'category').prefetch_related('tags', 'comments')

        return queryset

    def list(self, request: Request, *args, **kwargs) -> Response:
        """게시글 목록 조회"""
        # 쿼리 파라미터 처리
        author_id = request.query_params.get('author_id')

        # Service 호출
        if author_id:
            articles = self.service.get_user_articles(int(author_id))
        else:
            articles = self.service.get_published_articles()

        # Serialization
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)

    def create(self, request: Request, *args, **kwargs) -> Response:
        """게시글 생성"""
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Service 호출
        article = self.service.create_article(
            title=serializer.validated_data['title'],
            content=serializer.validated_data['content'],
            author=request.user,
            category_id=serializer.validated_data.get('category_id'),
            tag_ids=serializer.validated_data.get('tag_ids', [])
        )

        # 응답
        response_serializer = ArticleDetailSerializer(article, context={'request': request})
        return Response(response_serializer.data, status=status.HTTP_201_CREATED)

    def update(self, request: Request, *args, **kwargs) -> Response:
        """게시글 수정"""
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Service 호출
        article = self.service.update_article(
            article_id=kwargs['pk'],
            user=request.user,
            **serializer.validated_data
        )

        # 응답
        response_serializer = ArticleDetailSerializer(article, context={'request': request})
        return Response(response_serializer.data)

    def destroy(self, request: Request, *args, **kwargs) -> Response:
        """게시글 삭제"""
        self.service.delete_article(
            article_id=kwargs['pk'],
            user=request.user
        )
        return Response(status=status.HTTP_204_NO_CONTENT)

    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request: Request, pk=None) -> Response:
        """게시글 발행"""
        article = self.service.publish_article(
            article_id=pk,
            user=request.user
        )

        serializer = ArticleDetailSerializer(article, context={'request': request})
        return Response(serializer.data)

    @action(detail=False, methods=['get'])
    def my_articles(self, request: Request) -> Response:
        """내 게시글 목록"""
        articles = self.service.get_user_articles(request.user.id)
        serializer = ArticleListSerializer(articles, many=True, context={'request': request})
        return Response(serializer.data)
```

### 3. Generic Views 활용
```python
# views/comment_views.py
from rest_framework import generics
from rest_framework.permissions import IsAuthenticated

from apps.comments.models import Comment
from apps.comments.serializers import CommentSerializer, CommentCreateSerializer
from apps.comments.services.comment_service import CommentService

class CommentListCreateView(generics.ListCreateAPIView):
    """댓글 목록 조회 및 생성"""
    permission_classes = [IsAuthenticated]
    service_class = CommentService

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.service = self.service_class()

    def get_serializer_class(self):
        if self.request.method == 'POST':
            return CommentCreateSerializer
        return CommentSerializer

    def get_queryset(self):
        article_id = self.kwargs.get('article_id')
        return Comment.objects.filter(
            article_id=article_id
        ).select_related('author')

    def perform_create(self, serializer):
        """생성 시 Service 호출"""
        article_id = self.kwargs.get('article_id')
        self.service.create_comment(
            article_id=article_id,
            user=self.request.user,
            content=serializer.validated_data['content']
        )

class CommentDetailView(generics.RetrieveUpdateDestroyAPIView):
    """댓글 상세 조회, 수정, 삭제"""
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticated]
```

### 4. 페이지네이션
```python
# views/article_viewset.py
from rest_framework.pagination import PageNumberPagination

class ArticlePageNumberPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class ArticleViewSet(viewsets.ModelViewSet):
    pagination_class = ArticlePageNumberPagination
    # ...
```

### 5. 필터링과 검색
```python
# views/article_viewset.py
from django_filters import rest_framework as filters
from rest_framework.filters import SearchFilter, OrderingFilter

class ArticleFilter(filters.FilterSet):
    """게시글 필터"""
    title = filters.CharFilter(lookup_expr='icontains')
    author_id = filters.NumberFilter(field_name='author__id')
    created_after = filters.DateTimeFilter(field_name='created_at', lookup_expr='gte')
    created_before = filters.DateTimeFilter(field_name='created_at', lookup_expr='lte')

    class Meta:
        model = Article
        fields = ['title', 'author_id', 'is_published']

class ArticleViewSet(viewsets.ModelViewSet):
    filter_backends = [filters.DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = ArticleFilter
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'updated_at', 'title']
    ordering = ['-created_at']
    # ...
```

### 6. 에러 핸들링
```python
# views/article_viewset.py
from rest_framework.exceptions import NotFound, PermissionDenied, ValidationError
from apps.articles.exceptions import ArticleNotFound, ArticlePermissionDenied

class ArticleViewSet(viewsets.ModelViewSet):
    def update(self, request: Request, *args, **kwargs) -> Response:
        try:
            serializer = self.get_serializer(data=request.data)
            serializer.is_valid(raise_exception=True)

            article = self.service.update_article(
                article_id=kwargs['pk'],
                user=request.user,
                **serializer.validated_data
            )

            response_serializer = ArticleDetailSerializer(article, context={'request': request})
            return Response(response_serializer.data)

        except ArticleNotFound as e:
            raise NotFound(str(e))
        except ArticlePermissionDenied as e:
            raise PermissionDenied(str(e))
        except Exception as e:
            # 로깅
            logger.error(f"Failed to update article: {str(e)}")
            raise
```

### 7. Custom Actions
```python
class ArticleViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def like(self, request: Request, pk=None) -> Response:
        """게시글 좋아요"""
        self.service.like_article(
            article_id=pk,
            user=request.user
        )
        return Response({'message': 'Liked'}, status=status.HTTP_200_OK)

    @action(detail=True, methods=['delete'])
    def unlike(self, request: Request, pk=None) -> Response:
        """게시글 좋아요 취소"""
        self.service.unlike_article(
            article_id=pk,
            user=request.user
        )
        return Response({'message': 'Unliked'}, status=status.HTTP_200_OK)

    @action(detail=False, methods=['get'])
    def trending(self, request: Request) -> Response:
        """인기 게시글"""
        articles = self.service.get_trending_articles(limit=10)
        serializer = ArticleListSerializer(articles, many=True, context={'request': request})
        return Response(serializer.data)

    @action(detail=True, methods=['get'])
    def export(self, request: Request, pk=None) -> Response:
        """게시글 내보내기 (PDF, Markdown 등)"""
        format_type = request.query_params.get('format', 'pdf')
        file_content = self.service.export_article(
            article_id=pk,
            format_type=format_type
        )

        response = Response(file_content, content_type='application/pdf')
        response['Content-Disposition'] = f'attachment; filename="article_{pk}.pdf"'
        return response
```

### 8. 비동기 Views (Django 4.1+)
```python
# views/async_article_views.py
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from asgiref.sync import sync_to_async

from apps.articles.services.async_article_service import AsyncArticleService
from apps.articles.serializers import ArticleDetailSerializer

class AsyncArticleDetailView(APIView):
    """비동기 게시글 상세 조회"""

    async def get(self, request: Request, pk: int) -> Response:
        service = AsyncArticleService()

        # 비동기로 데이터 조회
        article_data = await service.get_article_with_related_data(pk)

        if not article_data:
            return Response(
                {'error': 'Article not found'},
                status=status.HTTP_404_NOT_FOUND
            )

        # Serialization (sync)
        serializer = await sync_to_async(ArticleDetailSerializer)(
            article_data['article'],
            context={'request': request}
        )

        response_data = await sync_to_async(lambda: serializer.data)()

        return Response(response_data)
```

### 9. API Versioning
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning',
}

# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter

router_v1 = DefaultRouter()
router_v1.register(r'articles', ArticleViewSetV1, basename='article')

router_v2 = DefaultRouter()
router_v2.register(r'articles', ArticleViewSetV2, basename='article')

urlpatterns = [
    path('api/v1/', include((router_v1.urls, 'v1'))),
    path('api/v2/', include((router_v2.urls, 'v2'))),
]

# views/v1/article_viewset.py
class ArticleViewSetV1(viewsets.ModelViewSet):
    """API v1 - 구버전"""
    # ...

# views/v2/article_viewset.py
class ArticleViewSetV2(viewsets.ModelViewSet):
    """API v2 - 신버전 (새로운 필드 추가 등)"""
    # ...
```

### 10. Rate Limiting (Throttling)
```python
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle

class ArticleCreateRateThrottle(UserRateThrottle):
    rate = '10/hour'  # 시간당 10개 제한

class ArticleViewSet(viewsets.ModelViewSet):
    throttle_classes = [UserRateThrottle, AnonRateThrottle]

    def get_throttles(self):
        """액션별 throttle 설정"""
        if self.action == 'create':
            return [ArticleCreateRateThrottle()]
        return super().get_throttles()
```

## 베스트 프랙티스

### 1. 얇은 Views 유지
```python
# ✅ 좋은 예 - View는 HTTP 처리만
class ArticleViewSet(viewsets.ModelViewSet):
    def create(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 비즈니스 로직은 Service로
        article = self.service.create_article(**serializer.validated_data)

        response_serializer = ArticleDetailSerializer(article)
        return Response(response_serializer.data, status=status.HTTP_201_CREATED)

# ❌ 나쁜 예 - View에 비즈니스 로직 포함
class ArticleViewSet(viewsets.ModelViewSet):
    def create(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # View에서 직접 비즈니스 로직 처리 (나쁨)
        article = Article.objects.create(**serializer.validated_data)
        article.tags.set(request.data.get('tag_ids', []))
        send_email(article.author.email, 'New article created')
        notify_subscribers(article)

        return Response(ArticleDetailSerializer(article).data)
```

### 2. 명확한 응답 상태 코드
```python
class ArticleViewSet(viewsets.ModelViewSet):
    def create(self, request):
        # 201 Created
        return Response(data, status=status.HTTP_201_CREATED)

    def update(self, request, pk):
        # 200 OK
        return Response(data, status=status.HTTP_200_OK)

    def destroy(self, request, pk):
        # 204 No Content
        return Response(status=status.HTTP_204_NO_CONTENT)

    @action(detail=True, methods=['post'])
    def custom_action(self, request, pk):
        # 202 Accepted (비동기 처리)
        return Response({'message': 'Processing'}, status=status.HTTP_202_ACCEPTED)
```

## 주의사항

### ❌ 피해야 할 패턴

1. **View에서 직접 모델 조작**
```python
# ❌ 나쁜 예
def create(self, request):
    article = Article.objects.create(...)
    article.tags.set(...)

# ✅ 좋은 예
def create(self, request):
    article = self.service.create_article(...)
```

2. **복잡한 쿼리를 View에서 작성**
```python
# ❌ 나쁜 예
def get_queryset(self):
    return Article.objects.filter(
        # 복잡한 필터링 로직
    ).annotate(
        # 복잡한 계산
    ).select_related(...).prefetch_related(...)

# ✅ 좋은 예 - Service나 Manager로 이동
def get_queryset(self):
    return self.service.get_optimized_queryset()
```

3. **에러를 무시하거나 일반적인 메시지 반환**
```python
# ❌ 나쁜 예
try:
    article = self.service.update_article(...)
except Exception:
    return Response({'error': 'Something went wrong'})

# ✅ 좋은 예
try:
    article = self.service.update_article(...)
except ArticleNotFound as e:
    raise NotFound(str(e))
except ArticlePermissionDenied as e:
    raise PermissionDenied(str(e))
```

## 체크리스트

- [ ] ViewSet 또는 Generic Views를 사용하고 있는가?
- [ ] 비즈니스 로직을 Service로 분리했는가?
- [ ] 적절한 Serializer를 액션별로 사용하는가?
- [ ] 권한과 인증이 올바르게 설정되어 있는가?
- [ ] 페이지네이션을 적용했는가?
- [ ] 적절한 HTTP 상태 코드를 반환하는가?
- [ ] 쿼리 최적화가 되어 있는가?
- [ ] 에러 핸들링이 명확한가?
- [ ] API 버저닝을 고려했는가?
- [ ] Rate limiting을 적용했는가?
