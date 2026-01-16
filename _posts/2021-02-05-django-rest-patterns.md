---
layout: blog
title: "Django REST Framework Best Practices: Building Scalable, Secure APIs"
description: "Architectural patterns and pitfalls to avoid when building production APIs with Django REST Framework."
date: 2021-02-05
categories: [backend, django, architecture]
tags: [django-rest-framework, api-design, scalability, security, python]
read_time: true
toc: true
toc_sticky: true
classes: wide
---


## The Problem: Not All REST APIs Are Created Equal

When we started building APIs with Django REST Framework (DRF), we made classic mistakes:

- **Serializers that serialize everything** - Exposing internal fields
- **Views that do too much** - Mixed authentication, validation, business logic
- **N+1 queries everywhere** - Loading related objects inefficiently
- **Weak pagination** - Cursor-based pagination only for large datasets
- **Inconsistent error responses** - Sometimes returning 400, sometimes 500
- **No request validation** - Trusting user input implicitly
- **Performance disasters** - Unbounded queries returning millions of rows

This article documents what we learned building APIs at scale using Django REST Framework.

---

## Architecture: Layered API Design

We adopted a three-layer architecture:

```
┌─────────────────────────────────────────────────┐
│    View Layer (APIView, ViewSet, Permissions)   │
│  - Route requests to handlers                    │
│  - Apply authentication & permissions            │
│  - Handle HTTP status codes                      │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│    Service Layer (Business Logic)               │
│  - Queries with filters/pagination              │
│  - Data transformation                          │
│  - External service integration                 │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│  Serializer Layer (Validation & Transformation) │
│  - Input validation                             │
│  - Output transformation                        │
│  - Related object handling                      │
└─────────────────────────────────────────────────┘
```

---

## Layer 1: Intelligent Serializers

### Problem: Over-Serialization

```python
# BAD: Serializes everything, including internal fields
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'  # Exposes password_hash, internal_id, etc!

# Usage
user = User.objects.get(id=1)
serializer = UserSerializer(user)
print(serializer.data)
# Output includes: password_hash, last_login, is_staff, is_superuser, etc.
```

### Solution: Explicit Fields with Context Awareness

```python
class UserSerializer(serializers.ModelSerializer):
    """Serialize user data with context-aware fields."""
    
    # Explicitly define what to expose
    full_name = serializers.SerializerMethodField()
    is_self = serializers.SerializerMethodField()
    
    class Meta:
        model = User
        fields = ['id', 'email', 'full_name', 'is_self', 'created_at']
        read_only_fields = ['id', 'created_at']
    
    def get_full_name(self, obj):
        """Computed field combining first + last name."""
        return f"{obj.first_name} {obj.last_name}".strip()
    
    def get_is_self(self, obj):
        """Only show if requesting user is viewing their own profile."""
        request = self.context.get('request')
        if not request:
            return False
        return request.user == obj

# Usage
serializer = UserSerializer(
    user,
    context={'request': request}  # Pass request for context
)
print(serializer.data)
# Output: {'id': 1, 'email': 'user@example.com', 'full_name': 'John Doe', 'is_self': true}
```

### Different Serializers for Different Contexts

```python
class UserListSerializer(serializers.ModelSerializer):
    """Minimal serializer for list views (faster, less data)."""
    
    class Meta:
        model = User
        fields = ['id', 'email', 'created_at']

class UserDetailSerializer(serializers.ModelSerializer):
    """Full serializer for detail views."""
    
    team = serializers.SerializerMethodField()
    permissions = serializers.SerializerMethodField()
    
    class Meta:
        model = User
        fields = ['id', 'email', 'full_name', 'team', 'permissions', 'created_at']
    
    def get_team(self, obj):
        # Only fetch related data in detail view
        team_serializer = TeamSerializer(obj.team)
        return team_serializer.data
    
    def get_permissions(self, obj):
        # Expensive permission calculation
        return obj.get_all_permissions()

class UserCreateSerializer(serializers.ModelSerializer):
    """Specialized serializer for creating users with validation."""
    
    password = serializers.CharField(write_only=True, min_length=8)
    password_confirm = serializers.CharField(write_only=True)
    
    class Meta:
        model = User
        fields = ['email', 'password', 'password_confirm', 'first_name', 'last_name']
    
    def validate(self, data):
        """Validate password confirmation."""
        if data['password'] != data.pop('password_confirm'):
            raise serializers.ValidationError("Passwords don't match")
        return data
    
    def create(self, validated_data):
        """Hash password before saving."""
        password = validated_data.pop('password')
        user = User(**validated_data)
        user.set_password(password)
        user.save()
        return user

# In views.py
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    
    def get_serializer_class(self):
        """Use appropriate serializer based on action."""
        if self.action == 'list':
            return UserListSerializer
        elif self.action == 'create':
            return UserCreateSerializer
        else:  # retrieve, update, destroy
            return UserDetailSerializer
```

---

## Layer 2: Optimized Views

### The N+1 Query Problem

```python
# BAD: N+1 queries
class OrderListView(APIView):
    def get(self, request):
        orders = Order.objects.all()  # 1 query
        
        result = []
        for order in orders:
            data = {
                'id': order.id,
                'customer': order.customer.name,  # N queries! (1 per order)
                'items': order.items.count(),  # N queries! (1 per order)
                'total': sum(item.price for item in order.items.all())  # N queries!
            }
            result.append(data)
        
        return Response(result)

# For 100 orders, this does: 1 + 100 + 100 + 100 = 301 queries! 🤦

# GOOD: Prefetch and select related
class OrderListView(APIView):
    def get(self, request):
        orders = Order.objects.select_related(
            'customer'  # JOIN customer table
        ).prefetch_related(
            'items__product'  # Separate query, but batched
        ).all()
        
        # Now we have at most 3 queries total:
        # 1. Load orders with customer (LEFT JOIN)
        # 2. Load items for all orders
        # 3. Load products for all items
        
        result = []
        for order in orders:
            data = {
                'id': order.id,
                'customer': order.customer.name,  # No query! Already loaded
                'items': order.items.count(),  # No query! Already loaded
                'total': sum(item.price for item in order.items.all())  # No query!
            }
            result.append(data)
        
        return Response(result)
```

### ViewSet with Optimized Queries

```python
class OrderViewSet(viewsets.ModelViewSet):
    """API endpoints for orders with optimized queries."""
    
    def get_queryset(self):
        """Return optimized queryset based on action."""
        queryset = Order.objects.all()
        
        if self.action == 'list':
            # List view: minimal data needed
            return queryset.select_related('customer').only(
                'id', 'customer__name', 'total_amount', 'created_at'
            )
        
        elif self.action == 'retrieve':
            # Detail view: include all related data
            return queryset.select_related(
                'customer',
                'shipping_address'
            ).prefetch_related(
                'items__product',
                'payments'
            )
        
        elif self.action == 'update':
            # Update doesn't need customer info
            return queryset.select_related('customer')
        
        return queryset
    
    def get_serializer_class(self):
        """Use different serializers for different actions."""
        if self.action == 'list':
            return OrderListSerializer
        elif self.action == 'create':
            return OrderCreateSerializer
        else:
            return OrderDetailSerializer
    
    def filter_queryset(self, queryset):
        """Apply filters and pagination."""
        queryset = super().filter_queryset(queryset)
        
        # Custom filtering
        status = self.request.query_params.get('status')
        if status:
            queryset = queryset.filter(status=status)
        
        customer_id = self.request.query_params.get('customer_id')
        if customer_id:
            queryset = queryset.filter(customer_id=customer_id)
        
        return queryset
```

---

## Layer 3: Pagination and Filtering

### Smart Pagination

```python
from rest_framework.pagination import CursorPagination, PageNumberPagination

class StandardPageNumberPagination(PageNumberPagination):
    """Offset/limit pagination for small datasets."""
    page_size = 100
    page_size_query_param = 'page_size'
    max_page_size = 1000

class StandardCursorPagination(CursorPagination):
    """Cursor pagination for large datasets (more efficient)."""
    page_size = 100
    ordering = '-created_at'  # Must have consistent ordering
    template = None

# Choose based on dataset size
class OrderViewSet(viewsets.ModelViewSet):
    def get_pagination_class(self):
        """Use cursor pagination for large datasets."""
        total = Order.objects.count()
        
        if total > 10000:
            return StandardCursorPagination
        else:
            return StandardPageNumberPagination
    
    def get_paginated_response(self, data):
        paginator = self.get_pagination_class()()
        return paginator.paginate_queryset(
            self.get_queryset(),
            self.request
        )
```

### Filtering

```python
from django_filters import rest_framework as filters

class OrderFilter(filters.FilterSet):
    """Custom filters for orders."""
    
    # Date range filtering
    created_after = filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='gte'
    )
    created_before = filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='lte'
    )
    
    # Text search
    customer_email = filters.CharFilter(
        field_name='customer__email',
        lookup_expr='icontains'
    )
    
    # Choice filter
    status = filters.ChoiceFilter(
        choices=Order.STATUS_CHOICES
    )
    
    class Meta:
        model = Order
        fields = ['status', 'customer_id']

class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    filter_backends = [filters.DjangoFilterBackend, filters.OrderingFilter]
    filterset_class = OrderFilter
    ordering_fields = ['created_at', 'total_amount']
    ordering = ['-created_at']  # Default ordering
```

---

## Error Handling & Validation

### Custom Exception Handler

```python
from rest_framework.exceptions import APIException
from rest_framework.response import Response
from rest_framework import status
import logging

logger = logging.getLogger(__name__)

class CustomException(APIException):
    """Base exception for API errors."""
    status_code = status.HTTP_400_BAD_REQUEST
    default_detail = "An error occurred"
    default_code = "error"

class OrderNotFoundError(CustomException):
    status_code = status.HTTP_404_NOT_FOUND
    default_detail = "Order not found"
    default_code = "order_not_found"

class InsufficientInventoryError(CustomException):
    status_code = status.HTTP_409_CONFLICT
    default_detail = "Insufficient inventory"
    default_code = "insufficient_inventory"

def custom_exception_handler(exc, context):
    """Handle exceptions with consistent format."""
    
    # Log unexpected errors
    if not isinstance(exc, APIException):
        logger.error(f"Unhandled exception: {str(exc)}", exc_info=True)
    
    # Get default response
    response = exception_handler(exc, context)
    
    if response is not None:
        # Standardize error response
        response.data = {
            'error': {
                'code': getattr(exc, 'default_code', 'error'),
                'message': str(exc.detail),
                'status': response.status_code
            }
        }
    
    return response

# In settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'api.exceptions.custom_exception_handler'
}
```

### Input Validation

```python
class OrderCreateSerializer(serializers.ModelSerializer):
    """Validate order creation."""
    
    class Meta:
        model = Order
        fields = ['customer_id', 'items', 'shipping_address']
    
    def validate_customer_id(self, value):
        """Validate customer exists."""
        try:
            Customer.objects.get(id=value)
        except Customer.DoesNotExist:
            raise serializers.ValidationError("Customer not found")
        return value
    
    def validate_items(self, value):
        """Validate order items."""
        if not value:
            raise serializers.ValidationError("Order must have at least one item")
        
        for item in value:
            # Check inventory
            product = item['product']
            qty = item['quantity']
            
            if product.stock < qty:
                raise serializers.ValidationError(
                    f"Insufficient inventory for {product.name}"
                )
        
        return value
    
    def validate(self, data):
        """Cross-field validation."""
        customer = Customer.objects.get(id=data['customer_id'])
        
        # Example: Check if customer can place orders
        if customer.is_blocked:
            raise serializers.ValidationError(
                "Customer account is blocked"
            )
        
        return data
```

---

## Authentication & Permissions

### Token Authentication with JWT

```python
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework.permissions import IsAuthenticated

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    """Add custom claims to JWT token."""
    
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # Add custom claims
        token['email'] = user.email
        token['first_name'] = user.first_name
        token['user_role'] = user.role
        
        return token

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        """Users only see their own orders."""
        return Order.objects.filter(customer__user=self.request.user)
```

### Custom Permissions

```python
from rest_framework.permissions import BasePermission

class IsOrderOwner(BasePermission):
    """Only allow order owner to view/edit."""
    
    def has_object_permission(self, request, view, obj):
        return obj.customer.user == request.user

class CanDeleteOrder(BasePermission):
    """Only allow deletion within 24 hours of creation."""
    
    def has_object_permission(self, request, view, obj):
        if request.method != 'DELETE':
            return True
        
        from datetime import timedelta
        from django.utils import timezone
        
        age = timezone.now() - obj.created_at
        return age < timedelta(hours=24)

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOrderOwner, CanDeleteOrder]
```

---

## Performance Monitoring

### View Performance Tracking

```python
import time
import logging
from rest_framework.decorators import api_view
from functools import wraps

logger = logging.getLogger(__name__)

def track_performance(func):
    """Decorator to track view performance."""
    
    @wraps(func)
    def wrapper(self, request, *args, **kwargs):
        start = time.time()
        
        try:
            response = func(self, request, *args, **kwargs)
            return response
        finally:
            duration = time.time() - start
            
            logger.info(f"API Request", extra={
                'method': request.method,
                'path': request.path,
                'status': getattr(response, 'status_code', 'unknown'),
                'duration_ms': duration * 1000,
                'user': request.user.id if request.user.is_authenticated else 'anonymous'
            })
    
    return wrapper

class OrderViewSet(viewsets.ModelViewSet):
    @track_performance
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
    
    @track_performance
    def retrieve(self, request, *args, **kwargs):
        return super().retrieve(request, *args, **kwargs)
```

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Not Documenting Your API

```python
# GOOD: Use drf-spectacular for auto-generated docs

# settings.py
INSTALLED_APPS = [
    ...
    'drf_spectacular',
]

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

# urls.py
urlpatterns = [
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema')),
]
```

### ❌ Mistake 2: Not Rate Limiting

```python
# GOOD: Add rate limiting

REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}
```

### ❌ Mistake 3: Forgetting CORS

```python
# GOOD: Configure CORS properly

INSTALLED_APPS = [
    'corsheaders',
    ...
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    ...
]

CORS_ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://admin.example.com",
]
```

---

## Implementation Checklist

- [ ] Design serializers for different contexts (list, detail, create)
- [ ] Add select_related/prefetch_related to all querysets
- [ ] Implement custom pagination for your use case
- [ ] Add filtering with django-filters
- [ ] Create custom exception handler
- [ ] Implement input validation in serializers
- [ ] Set up authentication (JWT or token-based)
- [ ] Define custom permissions
- [ ] Add view-level performance tracking
- [ ] Document API with drf-spectacular
- [ ] Add rate limiting
- [ ] Configure CORS correctly
- [ ] Set up API versioning if needed
- [ ] Test all endpoints with pytest
- [ ] Monitor API performance in production

---

## Key Takeaways

Building scalable Django REST APIs requires:

1. **Explicit serializers** - Different serializers for different use cases
2. **Optimized queries** - Use select_related/prefetch_related
3. **Smart pagination** - Cursor for large datasets, offset for small
4. **Strong validation** - Validate at serializer level
5. **Consistent errors** - Use custom exception handler
6. **Performance monitoring** - Track every request
7. **Documentation** - Auto-generate with drf-spectacular

The difference between a slow API and a fast one is often just optimization—the same features, but thoughtfully implemented.

---

## Related Resources

- [Django REST Framework Documentation](https://www.django-rest-framework.org/)
- [drf-spectacular for API Documentation](https://drf-spectacular.readthedocs.io/)
- [django-filters for Filtering](https://django-filter.readthedocs.io/)
- [SimpleJWT for Authentication](https://django-rest-framework-simplejwt.readthedocs.io/)
- [GitHub: Django REST Patterns](https://github.com/bharatkse/django-rest-patterns)

---

**Building APIs with Django REST?** What patterns do you use? Drop a comment below or reach out on [LinkedIn](https://linkedin.com/in/bharat-kumar28).
