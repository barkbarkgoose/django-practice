# Django REST API Practice Project

A Django REST API with JWT authentication for managing notes.

## Setup

```bash
# Create virtual environment
uv venv --python 3.13

# Activate virtual environment
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run migrations
python manage.py migrate

# Run server
python manage.py runserver
```

## API Endpoints

### Register User
```bash
curl -X POST http://localhost:8000/api/auth/register/ \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@example.com","password":"testpass123","password_confirm":"testpass123"}'
```

The response includes JWT tokens (access and refresh).

### Login
```bash
curl -X POST http://localhost:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"testpass123"}'
```

### Create Note
```bash
curl -X POST http://localhost:8000/api/notes/ \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{"title":"My Note","content":"Hello world"}'
```

### List Notes
```bash
curl -X GET http://localhost:8000/api/notes/ \
  -H "Authorization: Bearer <your_token>"
```

### Get Note
```bash
curl -X GET http://localhost:8000/api/notes/<note_id>/ \
  -H "Authorization: Bearer <your_token>"
```

### Update Note
```bash
curl -X PUT http://localhost:8000/api/notes/<note_id>/ \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{"title":"Updated Title","content":"Updated content"}'
```

### Delete Note
```bash
curl -X DELETE http://localhost:8000/api/notes/<note_id>/ \
  -H "Authorization: Bearer <your_token>"
```

## Extension Tasks

### 1. Add Filtering & Search
Add filtering and search capabilities to the notes endpoint:
- Filter notes by title using `?title=keyword`
- Search in content using `?search=keyword`
- Hint: Use `django_filters.rest_framework` and override `get_queryset` in ViewSet

### 2. Add Pagination
Configure pagination for the notes endpoint:
- Use `LimitOffsetPagination`
- Default limit: 10, Max limit: 100
- Hint: Configure in REST_FRAMEWORK settings

### 3. Add Ordering
Allow clients to order notes:
- `?ordering=created_at` or `?ordering=-created_at`
- `?ordering=updated_at` or `?ordering=-updated_at`
- Hint: Use `ordering_fields` and `ordering_default` in ViewSet

### 4. Add Custom Action
Add an archive endpoint for notes:
- `POST /api/notes/{id}/archive/`
- Add `archived` boolean field to Note model
- Hint: Use `@action(detail=True, methods=['post'])` decorator
