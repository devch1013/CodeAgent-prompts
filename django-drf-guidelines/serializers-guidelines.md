# Django REST Framework Serializers 가이드라인

## 개요
DRF Serializers 작성 시 따라야 할 최신 트렌드와 베스트 프랙티스를 제공합니다.

## 기본 원칙

### 1. Type Hints 사용
```python
from typing import Any, Dict, List, Optional
from rest_framework import serializers
from apps.articles.models import Article

class ArticleSerializer(serializers.ModelSerializer):
    author_name: str = serializers.CharField(source='author.username', read_only=True)

    class Meta:
        model = Article
        fields: List[str] = ['id', 'title', 'content', 'author_name', 'created_at']

    def validate_title(self, value: str) -> str:
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters")
        return value

    def create(self, validated_data: Dict[str, Any]) -> Article:
        return super().create(validated_data)
```

### 2. Read/Write 전용 Serializer 분리
```python
# ✅ 좋은 예 - 용도별 serializer 분리
class ArticleListSerializer(serializers.ModelSerializer):
    """목록 조회용 - 최소한의 필드만"""
    author_name = serializers.CharField(source='author.username', read_only=True)
    comment_count = serializers.IntegerField(read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'author_name', 'created_at', 'comment_count']

class ArticleDetailSerializer(serializers.ModelSerializer):
    """상세 조회용 - 모든 필드와 관계"""
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    tags = TagSerializer(many=True, read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'comments', 'tags', 'created_at', 'updated_at']

class ArticleCreateSerializer(serializers.ModelSerializer):
    """생성용 - 필요한 입력 필드만"""
    tag_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )

    class Meta:
        model = Article
        fields = ['title', 'content', 'tag_ids']

    def create(self, validated_data: Dict[str, Any]) -> Article:
        tag_ids = validated_data.pop('tag_ids', [])
        article = Article.objects.create(**validated_data)
        if tag_ids:
            article.tags.set(tag_ids)
        return article

class ArticleUpdateSerializer(serializers.ModelSerializer):
    """수정용"""
    class Meta:
        model = Article
        fields = ['title', 'content']
```

### 3. Nested Serializer 활용
```python
from typing import Dict, Any

class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email']

class CommentSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)

    class Meta:
        model = Comment
        fields = ['id', 'content', 'author', 'created_at']

class ArticleDetailSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'comments', 'created_at']
```

### 4. SerializerMethodField 활용
```python
from rest_framework import serializers
from typing import Any

class ArticleSerializer(serializers.ModelSerializer):
    author_full_name = serializers.SerializerMethodField()
    is_owner = serializers.SerializerMethodField()
    reading_time = serializers.SerializerMethodField()

    class Meta:
        model = Article
        fields = ['id', 'title', 'author_full_name', 'is_owner', 'reading_time']

    def get_author_full_name(self, obj: Article) -> str:
        return f"{obj.author.first_name} {obj.author.last_name}"

    def get_is_owner(self, obj: Article) -> bool:
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user
        return False

    def get_reading_time(self, obj: Article) -> int:
        """예상 읽기 시간 (분)"""
        words_per_minute = 200
        word_count = len(obj.content.split())
        return max(1, word_count // words_per_minute)
```

### 5. 유효성 검증
```python
from rest_framework import serializers
from typing import Dict, Any

class ArticleCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['title', 'content', 'category_id']

    def validate_title(self, value: str) -> str:
        """필드별 검증"""
        if Article.objects.filter(title__iexact=value).exists():
            raise serializers.ValidationError("This title already exists")
        return value

    def validate_content(self, value: str) -> str:
        """금지어 검증"""
        forbidden_words = ['spam', 'abuse']
        if any(word in value.lower() for word in forbidden_words):
            raise serializers.ValidationError("Content contains forbidden words")
        return value

    def validate(self, attrs: Dict[str, Any]) -> Dict[str, Any]:
        """전체 데이터 검증"""
        title = attrs.get('title', '')
        content = attrs.get('content', '')

        if title and content and title.lower() in content.lower():
            raise serializers.ValidationError({
                'title': 'Title should not be contained in content'
            })

        return attrs
```

### 6. 동적 필드 지원
```python
from typing import List, Optional, Any
from rest_framework import serializers

class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """
    필요한 필드만 선택적으로 반환하는 Serializer
    Usage: ArticleSerializer(article, fields=['id', 'title'])
    """

    def __init__(self, *args: Any, **kwargs: Any) -> None:
        fields: Optional[List[str]] = kwargs.pop('fields', None)
        exclude: Optional[List[str]] = kwargs.pop('exclude', None)

        super().__init__(*args, **kwargs)

        if fields is not None:
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)

        if exclude is not None:
            for field_name in exclude:
                self.fields.pop(field_name, None)

class ArticleSerializer(DynamicFieldsModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'created_at']

# 사용 예시
# ArticleSerializer(article, fields=['id', 'title'])
# ArticleSerializer(articles, many=True, exclude=['content'])
```

### 7. to_representation 커스터마이징
```python
from typing import Dict, Any
from rest_framework import serializers

class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'created_at']

    def to_representation(self, instance: Article) -> Dict[str, Any]:
        """출력 형식 커스터마이징"""
        data = super().to_representation(instance)

        # 날짜 포맷 커스터마이징
        request = self.context.get('request')
        if request and request.query_params.get('date_format') == 'short':
            data['created_at'] = instance.created_at.strftime('%Y-%m-%d')

        # 조건부 필드 추가
        if request and request.user == instance.author:
            data['is_owner'] = True

        return data
```

### 8. Write-Only 필드 처리
```python
class UserRegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'password_confirm']

    def validate(self, attrs: Dict[str, Any]) -> Dict[str, Any]:
        if attrs['password'] != attrs.pop('password_confirm'):
            raise serializers.ValidationError({
                'password_confirm': 'Passwords do not match'
            })
        return attrs

    def create(self, validated_data: Dict[str, Any]) -> User:
        password = validated_data.pop('password')
        user = User(**validated_data)
        user.set_password(password)
        user.save()
        return user
```

### 9. Related Field 처리
```python
from rest_framework import serializers

class ArticleCreateSerializer(serializers.ModelSerializer):
    # PrimaryKeyRelatedField - ID만 받기
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True
    )

    # SlugRelatedField - 특정 필드로 받기
    author_username = serializers.SlugRelatedField(
        slug_field='username',
        queryset=User.objects.all(),
        source='author',
        write_only=True
    )

    # ManyToMany
    tag_ids = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(),
        many=True,
        source='tags',
        write_only=True
    )

    class Meta:
        model = Article
        fields = ['title', 'content', 'category_id', 'author_username', 'tag_ids']
```

### 10. Context 활용
```python
from typing import Dict, Any
from rest_framework import serializers

class ArticleSerializer(serializers.ModelSerializer):
    is_bookmarked = serializers.SerializerMethodField()

    class Meta:
        model = Article
        fields = ['id', 'title', 'is_bookmarked']

    def get_is_bookmarked(self, obj: Article) -> bool:
        # context에서 현재 사용자 정보 가져오기
        request = self.context.get('request')
        if request and hasattr(request, 'user') and request.user.is_authenticated:
            return Bookmark.objects.filter(
                user=request.user,
                article=obj
            ).exists()
        return False

# views.py에서 사용
# serializer = ArticleSerializer(article, context={'request': request})
```

### 11. ListSerializer 커스터마이징
```python
from typing import List, Dict, Any
from rest_framework import serializers

class BulkUpdateListSerializer(serializers.ListSerializer):
    """Bulk update를 지원하는 ListSerializer"""

    def update(self, instance: List[Article], validated_data: List[Dict[str, Any]]) -> List[Article]:
        # id를 키로 매핑
        article_mapping = {article.id: article for article in instance}
        data_mapping = {item['id']: item for item in validated_data}

        # 업데이트 수행
        updated_articles = []
        for article_id, data in data_mapping.items():
            article = article_mapping.get(article_id)
            if article:
                article.title = data.get('title', article.title)
                article.content = data.get('content', article.content)
                updated_articles.append(article)

        # Bulk update
        Article.objects.bulk_update(updated_articles, ['title', 'content'])
        return updated_articles

class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content']
        list_serializer_class = BulkUpdateListSerializer
```

### 12. 성능 최적화
```python
from rest_framework import serializers
from django.db.models import Prefetch

class OptimizedArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    recent_comments = CommentSerializer(many=True, read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'recent_comments']

    @classmethod
    def get_optimized_queryset(cls, queryset):
        """최적화된 쿼리셋 반환"""
        return queryset.select_related(
            'author'
        ).prefetch_related(
            Prefetch(
                'comments',
                queryset=Comment.objects.select_related('author').order_by('-created_at')[:5],
                to_attr='recent_comments'
            )
        )

# views.py에서 사용
# queryset = OptimizedArticleSerializer.get_optimized_queryset(Article.objects.all())
# serializer = OptimizedArticleSerializer(queryset, many=True)
```

## 주의사항

### ❌ 피해야 할 패턴

1. **하나의 Serializer로 모든 작업 처리**
```python
# ❌ 나쁜 예
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = '__all__'  # 모든 필드 노출
```

2. **비즈니스 로직을 Serializer에 포함**
```python
# ❌ 나쁜 예
class OrderSerializer(serializers.ModelSerializer):
    def create(self, validated_data):
        order = super().create(validated_data)
        # 이메일 발송, 결제 처리 등 복잡한 로직
        send_email(order)
        process_payment(order)
        return order

# ✅ 좋은 예 - Service로 분리
class OrderSerializer(serializers.ModelSerializer):
    def create(self, validated_data):
        return super().create(validated_data)

# services/order_service.py에서 비즈니스 로직 처리
```

3. **N+1 쿼리 발생**
```python
# ❌ 나쁜 예
class ArticleSerializer(serializers.ModelSerializer):
    author_name = serializers.SerializerMethodField()

    def get_author_name(self, obj):
        return obj.author.username  # N+1 쿼리 발생

# ✅ 좋은 예
class ArticleSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.username', read_only=True)
    # 또는 views에서 select_related('author') 사용
```

## 체크리스트

- [ ] Type hints를 추가했는가?
- [ ] Read/Write 용도별로 serializer를 분리했는가?
- [ ] 필요한 필드만 노출하고 있는가?
- [ ] SerializerMethodField 대신 source를 사용할 수 있는가?
- [ ] 유효성 검증이 적절한 위치에 있는가?
- [ ] N+1 쿼리를 방지했는가?
- [ ] Context를 활용하여 동적 데이터를 제공하는가?
- [ ] Write-only 필드를 적절히 사용했는가?
- [ ] 비즈니스 로직을 service로 분리했는가?
- [ ] 성능 최적화를 고려했는가?
