# Django REST Framework Tutorial

This project is an implementation of the official Django REST Framework tutorial, demonstrating how to build RESTful APIs with Django.

## About

This project illustrates the fundamental concepts of Django REST Framework:
- Models and serialization
- Function-based and class-based views
- Authentication and permissions
- Relationships and hyperlinked APIs
- ViewSets and routers

## Setup

### Prerequisites

- Python 3.x
- [UV](https://github.com/astral-sh/uv) (Python package manager)
- [direnv](https://direnv.net/) (environment management)

### Installation

1. **Activate the environment**
   ```bash
   direnv allow
   ```

2. **Install dependencies**
   ```bash
   uv sync
   ```

3. **Apply migrations**
   ```bash
   python manage.py migrate
   ```

4. **Create a superuser** (optional)
   ```bash
   python manage.py createsuperuser
   ```

## Usage

### Install package
```bash
uv add pygment
```

### Start the development server
```bash
uv run python manage.py runserver
```

### Access the Django shell
```bash
python manage.py shell -v 2
```

## Project Structure

The project is organized as follows:
- `tutorial/` - Main project package
  - `snippets/` - Application for managing code snippets
    - `models.py` - Definition of the Snippet model
    - `serializers.py` - Serializers for converting between JSON and Python objects
    - `views.py` - Views for handling HTTP requests
    - `urls.py` - URL configuration for the API

## API Endpoints

- `GET /snippets/` - List all code snippets
- `POST /snippets/` - Create a new code snippet
- `GET /snippets/{id}/` - Retrieve a specific snippet
- `PUT /snippets/{id}/` - Update a specific snippet
- `DELETE /snippets/{id}/` - Delete a specific snippet

## Usage Examples

### Create a snippet via the API
```bash
curl -X POST http://localhost:8000/snippets/ -H "Content-Type: application/json" -d '{"title": "My snippet", "code": "print(\"Hello, world!\")", "language": "python"}'
```

### Retrieve the list of snippets
```bash
curl http://localhost:8000/snippets/
```

## Resources

- [Django Documentation](https://docs.djangoproject.com/)
- [Django REST Framework Documentation](https://www.django-rest-framework.org/)
- [Django REST Framework Tutorial](https://www.django-rest-framework.org/tutorial/quickstart/)