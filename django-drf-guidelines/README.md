# Django DRF 코드 가이드라인

## 소개

이 가이드라인은 Django REST Framework(DRF)를 사용한 프로젝트에서 따라야 할 2024-2025년 최신 트렌드와 베스트 프랙티스를 제공합니다. 각 영역별로 필요한 가이드라인을 독립적으로 참조할 수 있도록 구성되어 있습니다.

## 프로젝트 구조

```
{app_name}/
├── models/          # Database models
├── serializers/     # DRF serializers
├── services/        # Business logic
├── views/           # API views
├── exceptions.py    # Exception definitions
├── enums.py         # Enum classes
├── factories.py     # Test factories
└── urls.py          # URL routing
```

이 구조는 **관심사의 분리(Separation of Concerns)** 원칙을 따르며, 각 컴포넌트가 명확한 역할을 가지도록 설계되었습니다.

## 핵심 원칙

### 1. Service Layer 패턴
비즈니스 로직을 Views, Models, Serializers로부터 분리하여 Services에 캡슐화합니다.

- **Views**: HTTP 요청/응답 처리만 담당
- **Serializers**: 데이터 직렬화/역직렬화 및 입력 검증
- **Services**: 비즈니스 로직 및 복잡한 작업 처리
- **Models**: 데이터 구조 정의 및 간단한 유틸리티

### 2. Type Hints 사용
모든 코드에서 Python Type Hints를 사용하여 코드의 가독성과 유지보수성을 향상시킵니다.

### 3. Enum을 통한 상수 관리
매직 스트링/넘버를 방지하고 Enum을 사용하여 상태, 타입, 카테고리 등을 관리합니다.

### 4. 명확한 예외 처리
커스텀 예외를 정의하고 표준화된 에러 응답 형식을 사용합니다.

### 5. 테스트 자동화
Factory Pattern을 사용하여 테스트 데이터를 일관되게 생성하고 포괄적인 테스트를 작성합니다.

## 가이드라인 목록

각 가이드라인은 독립적으로 참조할 수 있으며, 필요한 영역의 가이드라인을 선택하여 프롬프트로 사용할 수 있습니다.

### 📊 [Models 가이드라인](./models-guidelines.md)
Django 모델 작성 시 따라야 할 가이드라인

**주요 내용:**
- Type Hints 사용
- Custom Manager와 QuerySet 활용
- Abstract Base Models
- Database 제약조건
- 쿼리 최적화 (select_related, prefetch_related)
- Enum을 활용한 상태 관리

**언제 사용하나요?**
- 새로운 모델을 작성할 때
- 모델 필드를 추가하거나 수정할 때
- 쿼리 성능을 최적화할 때

### 🔄 [Serializers 가이드라인](./serializers-guidelines.md)
DRF Serializer 작성 시 따라야 할 가이드라인

**주요 내용:**
- Type Hints 사용
- Read/Write 전용 Serializer 분리
- Nested Serializer 활용
- SerializerMethodField
- 유효성 검증
- 동적 필드 지원
- 성능 최적화

**언제 사용하나요?**
- API 입출력 데이터 형식을 정의할 때
- 데이터 검증 로직을 작성할 때
- N+1 쿼리를 방지하고 싶을 때

### ⚙️ [Services 가이드라인](./services-guidelines.md)
비즈니스 로직을 Service Layer로 분리하는 가이드라인

**주요 내용:**
- BaseService 패턴
- Transaction 관리
- 복잡한 비즈니스 로직 캡슐화
- Service 조합 (Composition)
- 비동기 Service (Async Support)
- 외부 의존성 주입

**언제 사용하나요?**
- 복잡한 비즈니스 로직을 작성할 때
- 여러 모델을 조합한 작업을 수행할 때
- 재사용 가능한 로직을 만들 때
- 테스트하기 쉬운 구조를 원할 때

### 🌐 [Views 가이드라인](./views-guidelines.md)
DRF Views/ViewSets 작성 시 따라야 할 가이드라인

**주요 내용:**
- ViewSet 패턴 (권장)
- Generic Views 활용
- 페이지네이션
- 필터링과 검색
- 에러 핸들링
- Custom Actions
- API Versioning
- Rate Limiting

**언제 사용하나요?**
- API 엔드포인트를 정의할 때
- HTTP 요청/응답을 처리할 때
- 권한과 인증을 설정할 때

### ⚠️ [Exceptions 가이드라인](./exceptions-guidelines.md)
예외 처리 및 에러 응답 가이드라인

**주요 내용:**
- Custom Exception 정의
- Exception 계층 구조
- 표준화된 에러 응답 형식
- Custom Exception Handler
- HTTP 상태 코드 활용
- 에러 로깅

**언제 사용하나요?**
- 비즈니스 로직에서 예외를 발생시킬 때
- API 에러 응답 형식을 표준화할 때
- 디버깅과 모니터링을 개선하고 싶을 때

### 🔢 [Enums 가이드라인](./enums-guidelines.md)
Enum을 활용한 상수 관리 가이드라인

**주요 내용:**
- Django TextChoices/IntegerChoices
- 비즈니스 로직을 가진 Enum
- 상태 머신 구현
- 다국어 지원
- DRF Serializer와 Enum 통합

**언제 사용하나요?**
- 상태, 타입, 카테고리 등의 고정 값을 정의할 때
- 매직 스트링/넘버를 제거하고 싶을 때
- 상태 전환 로직을 명확하게 하고 싶을 때

### 🏭 [Factories 가이드라인](./factories-guidelines.md)
테스트 데이터 생성을 위한 Factory Pattern 가이드라인

**주요 내용:**
- Factory Boy 활용
- 관계가 있는 Factory
- Trait를 활용한 변형
- Faker를 활용한 현실적인 데이터
- Fixture와 Factory 조합

**언제 사용하나요?**
- 테스트 코드를 작성할 때
- 일관된 테스트 데이터를 생성하고 싶을 때
- 복잡한 객체 관계를 쉽게 생성하고 싶을 때

### 🔗 [URLs 가이드라인](./urls-guidelines.md)
RESTful API URL 구조 가이드라인

**주요 내용:**
- ViewSet과 Router 활용
- Nested Routes
- Custom Actions
- API Versioning
- RESTful 설계 원칙
- 쿼리 파라미터 활용

**언제 사용하나요?**
- API URL 구조를 설계할 때
- RESTful 원칙을 따르고 싶을 때
- API 버저닝을 구현할 때

## 빠른 시작

### 1. 새로운 앱 생성
```bash
python manage.py startapp your_app
```

### 2. 디렉토리 구조 생성
```bash
cd your_app
mkdir models serializers services views
touch enums.py exceptions.py factories.py
```

### 3. 각 파일 초기화
```python
# models/__init__.py
from .user import User
from .article import Article

# serializers/__init__.py
from .user_serializers import UserSerializer
from .article_serializers import ArticleListSerializer, ArticleDetailSerializer

# services/__init__.py
from .user_service import UserService
from .article_service import ArticleService

# views/__init__.py
from .user_viewset import UserViewSet
from .article_viewset import ArticleViewSet
```

### 4. 가이드라인 참조
필요한 영역의 가이드라인을 참조하여 코드를 작성합니다.

## 개발 워크플로우

### 새로운 기능 추가 시

1. **모델 정의** → [Models 가이드라인](./models-guidelines.md) 참조
2. **Enum 정의** → [Enums 가이드라인](./enums-guidelines.md) 참조
3. **예외 정의** → [Exceptions 가이드라인](./exceptions-guidelines.md) 참조
4. **Serializer 작성** → [Serializers 가이드라인](./serializers-guidelines.md) 참조
5. **Service 작성** → [Services 가이드라인](./services-guidelines.md) 참조
6. **View 작성** → [Views 가이드라인](./views-guidelines.md) 참조
7. **URL 등록** → [URLs 가이드라인](./urls-guidelines.md) 참조
8. **Factory 작성** → [Factories 가이드라인](./factories-guidelines.md) 참조
9. **테스트 작성**

## 전체 예시

간단한 블로그 API를 만드는 전체 예시:

### 1. Enum 정의
```python
# enums.py
from django.db import models

class ArticleStatus(models.TextChoices):
    DRAFT = 'draft', 'Draft'
    PUBLISHED = 'published', 'Published'
    ARCHIVED = 'archived', 'Archived'
```

### 2. Exception 정의
```python
# exceptions.py
from rest_framework.exceptions import APIException

class ArticleNotFound(APIException):
    status_code = 404
    default_detail = 'Article not found.'
```

### 3. Model 정의
```python
# models/article.py
from django.db import models
from apps.blog.enums import ArticleStatus

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    status = models.CharField(
        max_length=20,
        choices=ArticleStatus.choices,
        default=ArticleStatus.DRAFT
    )
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
```

### 4. Serializer 정의
```python
# serializers/article_serializers.py
from rest_framework import serializers

class ArticleListSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'status', 'created_at']

class ArticleCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['title', 'content']
```

### 5. Service 정의
```python
# services/article_service.py
from apps.blog.exceptions import ArticleNotFound

class ArticleService:
    def create_article(self, title: str, content: str, author) -> Article:
        return Article.objects.create(
            title=title,
            content=content,
            author=author
        )

    def get_article(self, article_id: int) -> Article:
        try:
            return Article.objects.get(id=article_id)
        except Article.DoesNotExist:
            raise ArticleNotFound(article_id=article_id)
```

### 6. View 정의
```python
# views/article_viewset.py
from rest_framework import viewsets

class ArticleViewSet(viewsets.ModelViewSet):
    service_class = ArticleService

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.service = self.service_class()

    def create(self, request):
        serializer = ArticleCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        article = self.service.create_article(
            title=serializer.validated_data['title'],
            content=serializer.validated_data['content'],
            author=request.user
        )

        return Response(
            ArticleListSerializer(article).data,
            status=status.HTTP_201_CREATED
        )
```

### 7. URL 등록
```python
# urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')

urlpatterns = router.urls
```

### 8. Factory 정의
```python
# factories.py
import factory

class ArticleFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Faker('sentence')
    content = factory.Faker('paragraph')
    author = factory.SubFactory(UserFactory)
```

## 추가 리소스

### 공식 문서
- [Django Documentation](https://docs.djangoproject.com/)
- [Django REST Framework](https://www.django-rest-framework.org/)
- [Factory Boy](https://factoryboy.readthedocs.io/)

### 유용한 패키지
- `django-filter`: 필터링 지원
- `drf-spectacular`: OpenAPI 스키마 생성
- `django-cors-headers`: CORS 설정
- `djangorestframework-simplejwt`: JWT 인증
- `django-extensions`: 개발 도구

## 기여하기

이 가이드라인에 대한 피드백이나 개선 제안이 있다면 언제든지 제안해주세요!

## 라이선스

MIT License

---

**마지막 업데이트**: 2024-11

**버전**: 1.0.0
