# Django REST Framework Guidelines for Junie

This comprehensive guide provides essential concepts and code patterns for building REST APIs with Django REST Framework (DRF). It covers the complete progression from basic serialization to advanced ViewSets and routing.

## Initial Setup and Configuration

### Basic Project Setup
```python
# settings.py 
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'your_app_name',
]

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
}
```

### Recommended Project Structure
```
your_app/
├── models.py          # Django models
├── serializers.py     # DRF serializers
├── views.py          # API views
├── urls.py           # URL configuration
└── permissions.py    # Custom permissions
```

## Models and Serialization

### Model Creation with Metadata
```python
# models.py
from django.db import models
from django.contrib.auth.models import User
from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight

class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(max_length=100, default='python')
    style = models.CharField(max_length=100, default='friendly')
    owner = models.ForeignKey(User, related_name='snippets', on_delete=models.CASCADE)
    highlighted = models.TextField()

    class Meta:
        ordering = ['created']

    def save(self, *args, **kwargs):
        """Use pygments library to create highlighted HTML representation."""
        lexer = get_lexer_by_name(self.language)
        linenos = 'table' if self.linenos else False
        options = {'title': self.title} if self.title else {}
        formatter = HtmlFormatter(style=self.style, linenos=linenos, full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super().save(*args, **kwargs)
```

### Serializer Patterns

#### Explicit Serializer (Full Control)
```python
# serializers.py
from rest_framework import serializers
from .models import Snippet

class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.CharField(max_length=100, default='python')
    style = serializers.CharField(max_length=100, default='friendly')

    def create(self, validated_data):
        """Create and return a new Snippet instance."""
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """Update and return an existing Snippet instance."""
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

#### ModelSerializer (Recommended for Most Cases)
```python
class SnippetSerializer(serializers.ModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style', 'owner']

    def validate_code(self, value):
        """Custom field validation."""
        if 'malicious_code' in value:
            raise serializers.ValidationError("Code not allowed")
        return value
    
    def validate(self, data):
        """Cross-field validation."""
        if data['language'] == 'python' and not data['code'].strip():
            raise serializers.ValidationError("Python code cannot be empty")
        return data
```

#### HyperlinkedModelSerializer (For RESTful APIs)
```python
class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ['url', 'id', 'highlight', 'owner', 'title', 'code', 'linenos', 'language', 'style']

class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ['url', 'id', 'username', 'snippets']
```

## View Patterns and Progression

### 1. Function-Based Views with @api_view
```python
# views.py - Basic pattern
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET', 'POST'])
def snippet_list(request, format=None):
    """List all snippets or create a new snippet."""
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk, format=None):
    """Retrieve, update or delete a snippet."""
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### 2. Class-Based Views with APIView
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.http import Http404

class SnippetList(APIView):
    """List all snippets or create a new snippet."""
    
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class SnippetDetail(APIView):
    """Retrieve, update or delete a snippet instance."""
    
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### 3. Mixins and Generic Views
```python
from rest_framework import mixins, generics

# Using mixins for composable behavior
class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

# Generic class-based views (recommended)
class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

### 4. ViewSets (Highest Level of Abstraction)
```python
from rest_framework import viewsets, permissions, renderers
from rest_framework.decorators import action
from rest_framework.response import Response

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """ViewSet for read-only user operations."""
    queryset = User.objects.all()
    serializer_class = UserSerializer

class SnippetViewSet(viewsets.ModelViewSet):
    """ViewSet providing full CRUD operations + custom actions."""
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    @action(detail=False, methods=['post'])
    def set_favorite(self, request):
        """Custom action example."""
        # Custom logic here
        return Response({'status': 'favorite set'})

    def perform_create(self, serializer):
        """Automatically associate with authenticated user."""
        serializer.save(owner=self.request.user)
```

## Authentication and Permissions

### Built-in Permission Classes
```python
from rest_framework import permissions

class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
```

### Custom Permissions
```python
# permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """Custom permission to only allow owners to edit objects."""
    
    def has_object_permission(self, request, view, obj):
        # Read permissions for any request (GET, HEAD, OPTIONS)
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Write permissions only for the owner
        return obj.owner == request.user

# Usage in views
class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
```

### Automatic User Association
```python
class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def perform_create(self, serializer):
        """Automatically associate snippet with logged-in user."""
        serializer.save(owner=self.request.user)
```

## URL Configuration and Routing

### Manual URL Configuration
```python
# urls.py
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from . import views

urlpatterns = [
    path('', views.api_root),
    path('snippets/', views.SnippetList.as_view(), name='snippet-list'),
    path('snippets/<int:pk>/', views.SnippetDetail.as_view(), name='snippet-detail'),
    path('snippets/<int:pk>/highlight/', views.SnippetHighlight.as_view(), name='snippet-highlight'),
    path('users/', views.UserList.as_view(), name='user-list'),
    path('users/<int:pk>/', views.UserDetail.as_view(), name='user-detail'),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

### Automatic Routing with ViewSets
```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

# Create router and register ViewSets
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet, basename='snippet')
router.register(r'users', views.UserViewSet, basename='user')

# URLs determined automatically by the router
urlpatterns = [
    path('', include(router.urls)),
]
```

### API Root Endpoint
```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse

@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```

## Advanced Concepts

### Working with Relationships
```python
# Primary key relationships (default)
class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

# Hyperlinked relationships (RESTful)
class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(
        many=True, 
        view_name='snippet-detail', 
        read_only=True
    )
```

### Custom Actions in ViewSets
```python
from rest_framework.decorators import action

class SnippetViewSet(viewsets.ModelViewSet):
    # ... base configuration
    
    @action(detail=True, methods=['post'])
    def set_favorite(self, request, pk=None):
        """Custom action for individual objects."""
        snippet = self.get_object()
        # Custom logic
        return Response({'status': 'favorite set'})
    
    @action(detail=False)
    def recent_snippets(self, request):
        """Custom action for collection."""
        recent = Snippet.objects.filter(created__gte=timezone.now() - timedelta(days=1))
        serializer = self.get_serializer(recent, many=True)
        return Response(serializer.data)
```

### Content Negotiation and Format Suffixes
```python
# Views support multiple formats automatically
def snippet_list(request, format=None):
    # Will respond with JSON, HTML, or other formats based on:
    # 1. Accept header
    # 2. Format suffix (.json, .api)
    # 3. format query parameter
    pass
```

### Browsable API Configuration
```python
# project/urls.py
urlpatterns += [
    path('api-auth/', include('rest_framework.urls')),
]
```

## Key Concepts and When to Use Each Pattern

### Request and Response Objects
- **Request object**: Extends HttpRequest with `request.data` for flexible parsing
- **Response object**: Handles content negotiation automatically
- **Status codes**: Use named constants like `status.HTTP_201_CREATED`

### Serializer Concepts
- **Serialization**: Convert model instances to Python data types
- **Deserialization**: Convert incoming data to validated Python objects
- **Validation**: Field-level and object-level validation methods
- **many=True**: Use for querysets/lists of objects

### View Evolution Path
1. **@api_view decorators**: Simple function-based views
2. **APIView**: Class-based views with method handlers
3. **Generic views**: Pre-built CRUD operations
4. **ViewSets**: Group related views, automatic URL routing

### Authentication and Permission Layers
- **Authentication**: Who is making the request?
- **Permissions**: What can they do?
- **Object-level permissions**: Fine-grained control per object

## Testing Patterns
```python
from rest_framework.test import APITestCase
from rest_framework import status
from django.contrib.auth.models import User

class SnippetAPITestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='testuser', password='testpass')
        self.snippet = Snippet.objects.create(
            title='Test snippet',
            code='print("hello")',
            owner=self.user
        )
    
    def test_get_snippet_list(self):
        response = self.client.get('/snippets/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
    
    def test_create_snippet_authenticated(self):
        self.client.force_authenticate(user=self.user)
        data = {'title': 'New snippet', 'code': 'print("new")'}
        response = self.client.post('/snippets/', data)
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
```

## Best Practices and Decision Guidelines

### When to Use Each View Type:
- **Function-based views**: Simple, one-off logic
- **APIView**: Need full control over request handling
- **Generic views**: Standard CRUD operations (recommended)
- **ViewSets**: Consistent API with many endpoints

### Serializer Choice:
- **Serializer**: Custom logic, non-model data
- **ModelSerializer**: Standard model serialization (recommended)
- **HyperlinkedModelSerializer**: RESTful APIs with discoverability

### URL Patterns:
- **Manual URLs**: Explicit control, custom endpoints
- **Routers**: Consistent conventions, less code

This guide provides a complete foundation for building REST APIs with Django REST Framework, progressing from basic concepts to advanced patterns while maintaining best practices throughout.

Sources
[1] Tutorial 3: Class-based Views https://www.django-rest-framework.org/tutorial/3-class-based-views/
[2] Tutorial 1: Serialization https://www.django-rest-framework.org/tutorial/1-serialization/
[3] Tutorial 2: Requests and Responses https://www.django-rest-framework.org/tutorial/2-requests-and-responses/
[4] Tutorial 5: Relationships & Hyperlinked APIs https://www.django-rest-framework.org/tutorial/5-relationships-and-hyperlinked-apis/
[5] Tutorial 4: Authentication & Permissions https://www.django-rest-framework.org/tutorial/4-authentication-and-permissions/
[6] Tutorial 6: ViewSets & Routers https://www.django-rest-framework.org/tutorial/6-viewsets-and-routers/
