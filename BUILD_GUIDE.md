# Django REST API Practice Project - Build Guide

This document walks through the complete process of setting up a Django REST API project with JWT authentication, from scratch to a working state.

## Prerequisites

- Python 3.13 installed
- `uv` package manager installed (`brew install uv`)

---

## Step 1: Project Setup

### Create the project directory

```bash
mkdir -p django-practice && cd django-practice
```

### Create requirements.txt

```bash
# requirements.txt
Django==5.2
djangorestframework==3.15.2
djangorestframework-simplejwt==5.4.0
django-cors-headers==4.7.0
python-dotenv==1.0.1
```

**Note:** A `.python-version` file is not needed since we're explicitly specifying the Python version with `--python 3.13` in the venv command.

---

## Step 2: Virtual Environment & Dependencies

```bash
# Create uv virtual environment with Python 3.13
uv venv --python 3.13

# Activate the virtual environment
source .venv/bin/activate

# Install dependencies
uv pip install -r requirements.txt
```

---

## Step 3: Create Django Project

```bash
# Create the Django project (this generates manage.py and config/ package)
django-admin startproject config .

# Create the Django apps
python manage.py startapp apps/users
python manage.py startapp apps/notes
```

The `startproject` command creates `config/settings.py` with Django defaults. Add these imports at the top:

```python
from datetime import timedelta
from dotenv import load_dotenv

load_dotenv()
```

Then modify these settings in `config/settings.py`:

```python
# After INSTALLED_APPS, add:
INSTALLED_APPS = [
    # ... existing apps ...
    "rest_framework",
    "rest_framework_simplejwt",
    "corsheaders",
    "apps.users",
    "apps.notes",
]

# After MIDDLEWARE, add corsheaders:
MIDDLEWARE = [
    # ... existing middleware ...
    "corsheaders.middleware.CorsMiddleware",
]

# Update SECRET_KEY to use env var:
SECRET_KEY = os.getenv("SECRET_KEY", "django-insecure-dev-key")

# Update DEBUG:
DEBUG = os.getenv("DEBUG", "True") == "True"

# Add at the end:
AUTH_USER_MODEL = "users.User"

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": ("rest_framework.permissions.IsAuthenticated",),
}

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(hours=1),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=1),
    "ROTATE_REFRESH_TOKENS": False,
    "BLACKLIST_AFTER_ROTATION": True,
    "AUTH_HEADER_TYPES": ("Bearer",),
}

CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "http://localhost:8000",
]
```

The `startproject` command also creates `config/urls.py` and `config/wsgi.py`. Update `config/wsgi.py` to reference the correct settings module:

```python
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
application = get_wsgi_application()
```

Then update `config/urls.py` to include your app URLs:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/auth/", include("apps.users.urls")),
    path("api/notes/", include("apps.notes.urls")),
]
```

### Create .env file

```bash
# .env
SECRET_KEY=django-insecure-dev-key-change-in-production
DEBUG=True
```

---

## Step 4: Users App (Authentication)

### Create apps/__init__.py

```python
```

### Create apps/users/__init__.py

```python
```

### Create apps/users/apps.py

```python
from django.apps import AppConfig


class UsersConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.users"
```

### Create apps/users/models.py

```python
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    email = models.EmailField(unique=True)
```

### Create apps/users/serializers.py

```python
from rest_framework import serializers
from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password

User = get_user_model()


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ("id", "username", "email", "password")
        read_only_fields = ("id",)

    def create(self, validated_data):
        user = User.objects.create_user(
            username=validated_data["username"],
            email=validated_data.get("email", ""),
            password=validated_data["password"],
        )
        return user


class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, validators=[validate_password])
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ("username", "email", "password", "password_confirm")

    def validate(self, attrs):
        if attrs["password"] != attrs["password_confirm"]:
            raise serializers.ValidationError({"password": "Passwords don't match"})
        return attrs

    def create(self, validated_data):
        validated_data.pop("password_confirm")
        user = User.objects.create_user(
            username=validated_data["username"],
            email=validated_data.get("email", ""),
            password=validated_data["password"],
        )
        return user
```

### Create apps/users/views.py

```python
from rest_framework import generics, status
from rest_framework.response import Response
from rest_framework.permissions import AllowAny
from rest_framework_simplejwt.tokens import RefreshToken

from .serializers import RegisterSerializer, UserSerializer


class RegisterView(generics.CreateAPIView):
    serializer_class = RegisterSerializer
    permission_classes = [AllowAny]

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()

        refresh = RefreshToken.for_user(user)
        return Response(
            {
                "user": {
                    "id": user.id,
                    "username": user.username,
                    "email": user.email,
                },
                "refresh": str(refresh),
                "access": str(refresh.access_token),
            },
            status=status.HTTP_201_CREATED,
        )


class LoginView(generics.GenericAPIView):
    permission_classes = [AllowAny]

    def post(self, request, *args, **kwargs):
        username = request.data.get("username")
        password = request.data.get("password")

        from django.contrib.auth import authenticate
        user = authenticate(username=username, password=password)

        if user is None:
            return Response(
                {"error": "Invalid credentials"},
                status=status.HTTP_401_UNAUTHORIZED,
            )

        refresh = RefreshToken.for_user(user)
        return Response(
            {
                "user": {
                    "id": user.id,
                    "username": user.username,
                    "email": user.email,
                    "password": user.password,
                },
                "refresh": str(refresh),
                "access": str(refresh.access_token),
            }
        )
```

### Create apps/users/urls.py

```python
from django.urls import path
from .views import RegisterView, LoginView

urlpatterns = [
    path("register/", RegisterView.as_view(), name="register"),
    path("login/", LoginView.as_view(), name="login"),
]
```

---

## Step 5: Notes App (CRUD API)

### Create apps/notes/__init__.py

```python
```

### Create apps/notes/apps.py

```python
from django.apps import AppConfig


class NotesConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.notes"
```

### Create apps/notes/models.py

```python
from django.db import models
from django.conf import settings


class Note(models.Model):
    title = models.CharField(max_length=255)
    content = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    owner = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="notes",
    )

    def __str__(self):
        return self.title
```

### Create apps/notes/serializers.py

```python
from rest_framework import serializers
from .models import Note


class NoteSerializer(serializers.ModelSerializer):
    owner = serializers.StringRelatedField()

    class Meta:
        model = Note
        fields = ["id", "title", "content", "created_at", "updated_at", "owner"]
        read_only_fields = ["id", "created_at", "updated_at", "owner"]
```

### Create apps/notes/views.py

```python
from rest_framework import viewsets, permissions
from .models import Note
from .serializers import NoteSerializer


class NoteViewSet(viewsets.ModelViewSet):
    serializer_class = NoteSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(owner=self.request.user)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Create apps/notes/urls.py

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import NoteViewSet

router = DefaultRouter()
router.register("", NoteViewSet, basename="note")

urlpatterns = [
    path("", include(router.urls)),
]
```

---

## Step 6: Database Setup

### Update settings to use custom User model

In `config/settings/base.py`, ensure this line exists:
```python
AUTH_USER_MODEL = "users.User"
```

### Run initial migrations

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## Step 7: Testing the API

### Start the server

```bash
python manage.py runserver 8000
```

### Register a new user

```bash
curl -X POST http://localhost:8000/api/auth/register/ \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@example.com","password":"testpass123","password_confirm":"testpass123"}'
```

Response:
```json
{
  "user": {
    "id": 1,
    "username": "testuser",
    "email": "test@example.com"
  },
  "refresh": "<refresh_token>",
  "access": "<access_token>"
}
```

### Login

```bash
curl -X POST http://localhost:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass123"}'
```

### Create a note (requires authentication)

```bash
curl -X POST http://localhost:8000/api/notes/ \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"title":"My First Note","content":"Hello world from Django API!"}'
```

### List notes

```bash
curl -X GET http://localhost:8000/api/notes/ \
  -H "Authorization: Bearer <access_token>"
```

---

## Structure & Design Decisions

### Project Architecture

The project follows Django's recommended multi-app structure:

```
django-practice/
├── config/           # Django project configuration
│   └── settings/     # Settings split for different environments
├── apps/             # Django applications
│   ├── users/        # Authentication & user management
│   └── notes/        # Core business logic
├── manage.py         # Django management script
├── requirements.txt  # Python dependencies
└── .env              # Environment variables
```

### Key Design Decisions

#### 1. Custom User Model

We extend `AbstractUser` instead of using the default User model. This is a Django best practice that allows:
- Custom fields (e.g., email as unique identifier)
- Future extensibility without migrations
- Cleaner separation of concerns

#### 2. JWT Authentication with simplejwt

We use `djangorestframework-simplejwt` because:
- Industry standard for Django REST APIs
- Stateless authentication (good for APIs)
- Token refresh mechanism built-in
- Configurable token lifetimes

**Important Note:** The JWT settings must use `datetime.timedelta` objects, not strings. This is a common pitfall:
```python
# Correct
"ACCESS_TOKEN_LIFETIME": timedelta(hours=1)

# Incorrect (causes TypeError)
"ACCESS_TOKEN_LIFETIME": "1h"
```

#### 3. ViewSets for CRUD Operations

We use DRF's `ModelViewSet` because:
- Provides all CRUD operations out of the box
- Integrates with routers for clean URL configuration
- Easy to customize with custom actions

#### 4. Ownership-Based Querysets

The `NoteViewSet.get_queryset()` filters by `owner=request.user`, ensuring users can only see and modify their own notes. This is critical for security.

#### 5. IsAuthenticated Permission

We set `permission_classes = [permissions.IsAuthenticated]` on the ViewSet, requiring authentication for all note endpoints. The registration and login endpoints use `AllowAny` via `permission_classes = [AllowAny]`.

#### 6. SQLite for Development

Using SQLite simplifies development because:
- No separate database server required
- Zero configuration
- Easy to reset (just delete `db.sqlite3`)

### Why These Choices Matter for Interviews

1. **Custom User Model** - interviewers often ask about AUTH_USER_MODEL
2. **JWT Auth** - understanding tokens, refresh vs access, is fundamental
3. **ViewSets & Routers** - shows knowledge of DRF
4. **Ownership/permissions** - demonstrates security awareness
5. **Filtering querysets** - common real-world requirement

---

## Extension Tasks (for practice)

See README.md for 4 interview-focused extension tasks:
1. Filtering & Search
2. Pagination
3. Ordering
4. Custom Actions
