# Part 51: Regex in Web Frameworks — Flask, Django, FastAPI

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-50

---

## 51.1 URL Routing with Regex

```python
import re
from typing import Callable, Dict, Optional, Any

print("URL Routing with Regex:")
print("=" * 60)

# ===== Basic URL pattern matching =====
print("\n1. Simple URL router:")

class Router:
    """Simple regex-based URL router"""

    def __init__(self):
        self.routes: list = []

    def route(self, pattern: str, methods: list = None):
        """Decorator to register a route"""
        def decorator(fn: Callable):
            compiled = re.compile(f'^{pattern}$')
            self.routes.append({
                'pattern': compiled,
                'handler': fn,
                'methods': methods or ['GET'],
                'raw': pattern,
            })
            return fn
        return decorator

    def match(self, path: str, method: str = 'GET') -> Optional[Dict]:
        for route in self.routes:
            m = route['pattern'].match(path)
            if m and method in route['methods']:
                return {
                    'handler': route['handler'],
                    'kwargs': m.groupdict(),
                    'args': m.groups(),
                }
        return None

    def dispatch(self, path: str, method: str = 'GET') -> str:
        result = self.match(path, method)
        if result:
            return result['handler'](**result['kwargs'])
        return f"404 Not Found: {method} {path}"


# ===== Register routes =====
router = Router()

@router.route(r'/')
def index():
    return "Home Page"

@router.route(r'/users')
def user_list():
    return "User List"

@router.route(r'/users/(?P<user_id>\d+)')
def user_detail(user_id: str):
    return f"User #{user_id}"

@router.route(r'/users/(?P<user_id>\d+)/posts/(?P<post_id>\d+)')
def user_post(user_id: str, post_id: str):
    return f"User #{user_id}, Post #{post_id}"

@router.route(r'/articles/(?P<year>\d{4})/(?P<month>\d{2})/(?P<slug>[\w-]+)')
def article(year: str, month: str, slug: str):
    return f"Article: {year}/{month}/{slug}"

@router.route(r'/search', methods=['GET'])
def search():
    return "Search Page"

# ===== Test routing =====
paths = [
    ('GET',  '/'),
    ('GET',  '/users'),
    ('GET',  '/users/42'),
    ('GET',  '/users/42/posts/7'),
    ('GET',  '/articles/2024/10/my-first-post'),
    ('GET',  '/not-found'),
    ('POST', '/search'),  # wrong method
    ('GET',  '/search'),
]

print(f"\n   {'Method':<8} {'Path':<45} {'Result'}")
print(f"   {'-'*8} {'-'*45} {'-'*30}")
for method, path in paths:
    result = router.dispatch(path, method)
    print(f"   {method:<8} {path:<45} {result}")
```

---

## 51.2 Django URL Patterns

```python
import re

print("\nDjango-style URL Patterns:")
print("=" * 60)

# Django URL pattern syntax
# path(): simple patterns without regex
# re_path(): full regex patterns
# <type:name>: type converters

# ===== Type converters (like Django) =====
print("\n1. Django path() converter simulation:")

# Django converters: int, str, slug, uuid, path
CONVERTERS = {
    'int':  r'\d+',
    'str':  r'[^/]+',
    'slug': r'[-\w]+',
    'uuid': r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}',
    'path': r'.+',
}

def django_path_to_regex(path: str) -> re.Pattern:
    """Convert Django path() syntax to regex"""
    # Convert <type:name> to named group
    def replace_converter(m):
        conv_type = m.group(1)
        name = m.group(2)
        pattern = CONVERTERS.get(conv_type, r'[^/]+')
        return f'(?P<{name}>{pattern})'

    regex = re.sub(r'<(\w+):(\w+)>', replace_converter, path)
    # Simple <name> without type → str
    regex = re.sub(r'<(\w+)>', lambda m: f'(?P<{m.group(1)}>[^/]+)', regex)
    return re.compile(f'^/{regex}$')


django_patterns = [
    'users/',
    'users/<int:pk>/',
    'users/<int:pk>/profile/',
    'articles/<int:year>/<slug:slug>/',
    'files/<path:file_path>',
    'orders/<uuid:order_id>/',
]

print(f"\n   {'Django Pattern':<40} {'Regex'}")
print(f"   {'-'*40} {'-'*50}")
for dp in django_patterns:
    rx = django_path_to_regex(dp)
    print(f"   {dp:<40} {rx.pattern}")

# ===== Test converted patterns =====
print("\n2. Testing converted patterns:")
test_urls = [
    '/users/',
    '/users/42/',
    '/users/42/profile/',
    '/articles/2024/my-great-post/',
    '/files/docs/readme.txt',
    '/orders/550e8400-e29b-41d4-a716-446655440000/',
]

patterns_to_test = {
    'users/<int:pk>/':           'users/<int:pk>/',
    'articles/<int:year>/<slug:slug>/': 'articles/<int:year>/<slug:slug>/',
    'files/<path:file_path>':    'files/<path:file_path>',
    'orders/<uuid:order_id>/':   'orders/<uuid:order_id>/',
}

for url in test_urls:
    for name, dp in patterns_to_test.items():
        rx = django_path_to_regex(dp)
        m = rx.match(url)
        if m:
            print(f"\n   {url}")
            print(f"   → matches '{name}'")
            print(f"   → kwargs: {m.groupdict()}")
            break
```

---

## 51.3 FastAPI / Pydantic Validation with Regex

```python
import re
from dataclasses import dataclass, field
from typing import Optional

print("\nFastAPI-style Validation with Regex:")
print("=" * 60)

# ===== Field validators using regex =====
print("\n1. Input validation decorators:")

class ValidationError(ValueError):
    pass

def validate_regex(pattern: str, message: str = None):
    """Decorator factory: validate string field against regex"""
    compiled = re.compile(pattern)
    def validator(value: str) -> str:
        if not compiled.fullmatch(value):
            raise ValidationError(message or f"Value {value!r} doesn't match {pattern!r}")
        return value
    return validator

# Validators
validate_email    = validate_regex(
    r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
    "Invalid email format"
)
validate_phone_th = validate_regex(
    r'0[6-9]\d{8}',
    "Thai phone must be 10 digits starting with 06-09"
)
validate_thai_id  = validate_regex(
    r'\d{13}',
    "Thai ID must be 13 digits"
)
validate_postcode = validate_regex(
    r'\d{5}',
    "Postcode must be 5 digits"
)
validate_username = validate_regex(
    r'[a-zA-Z][a-zA-Z0-9_]{2,19}',
    "Username: 3-20 chars, start with letter, letters/digits/underscore"
)


# ===== User registration form =====
@dataclass
class UserRegistration:
    username: str
    email: str
    phone: str
    thai_id: str
    postcode: str

    def __post_init__(self):
        errors = {}
        fields_validators = {
            'username': validate_username,
            'email':    validate_email,
            'phone':    validate_phone_th,
            'thai_id':  validate_thai_id,
            'postcode': validate_postcode,
        }
        for fname, validator in fields_validators.items():
            try:
                setattr(self, fname, validator(getattr(self, fname)))
            except ValidationError as e:
                errors[fname] = str(e)
        if errors:
            raise ValidationError(f"Validation failed: {errors}")


# Test cases
test_users = [
    dict(username='alice_01', email='alice@example.com', phone='0891234567',
         thai_id='1234567890123', postcode='10110'),
    dict(username='bob', email='bob-invalid', phone='1234567890',
         thai_id='123', postcode='abc'),
    dict(username='2invalid', email='valid@email.com', phone='0821234567',
         thai_id='9876543210987', postcode='50000'),
]

print(f"\n   Registration tests:")
for i, user_data in enumerate(test_users, 1):
    print(f"\n   Test {i}: {user_data}")
    try:
        user = UserRegistration(**user_data)
        print(f"   ✓ Valid: {user.username} / {user.email}")
    except ValidationError as e:
        print(f"   ✗ Invalid: {e}")

# ===== Query parameter validation =====
print("\n\n2. Query parameter validation:")

QUERY_VALIDATORS = {
    'page':     re.compile(r'[1-9]\d*'),           # positive integer
    'per_page': re.compile(r'(?:10|25|50|100)'),   # allowed values
    'sort':     re.compile(r'(?:asc|desc)'),        # sort direction
    'q':        re.compile(r'[\w\s\-]{1,100}'),     # search query
    'from_date':re.compile(r'\d{4}-\d{2}-\d{2}'),  # ISO date
    'to_date':  re.compile(r'\d{4}-\d{2}-\d{2}'),  # ISO date
}

def validate_query_params(params: dict) -> dict:
    errors = {}
    validated = {}
    for key, value in params.items():
        if key in QUERY_VALIDATORS:
            if QUERY_VALIDATORS[key].fullmatch(str(value)):
                validated[key] = value
            else:
                errors[key] = f"Invalid value {value!r} for parameter '{key}'"
        else:
            errors[key] = f"Unknown parameter '{key}'"
    return validated, errors

test_queries = [
    {'page': '2', 'per_page': '25', 'sort': 'asc', 'q': 'python regex'},
    {'page': '0', 'per_page': '99', 'sort': 'random', 'unknown': 'x'},
    {'from_date': '2024-01-01', 'to_date': '2024-12-31', 'page': '1'},
]

print(f"\n   {'Query':<60} {'Status'}")
print(f"   {'-'*60} {'-'*30}")
for q in test_queries:
    valid, errors = validate_query_params(q)
    if errors:
        print(f"   {str(q)[:58]}")
        print(f"   → Errors: {errors}")
    else:
        print(f"   {str(q)[:58]}")
        print(f"   → OK: {valid}")
    print()
```

---

## 51.4 Middleware: Input Sanitization

```python
import re
from typing import Dict, Any

print("\nMiddleware: Input Sanitization:")
print("=" * 60)

# ===== Request sanitization =====
print("\n1. Request sanitizer middleware:")

class RequestSanitizer:
    """Sanitize and validate incoming request data"""

    # Patterns for dangerous content
    SQL_INJECT = re.compile(
        r"(?:--|\b(?:UNION|SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC"
        r"|EXECUTE|SCRIPT|XP_|SP_)\b)",
        re.IGNORECASE
    )

    XSS_PATTERNS = re.compile(
        r'(?:<script|javascript:|on\w+=|<iframe|<object|<embed|<svg\s)',
        re.IGNORECASE
    )

    PATH_TRAVERSAL = re.compile(r'\.{2}[/\\]|[/\\]\.{2}')

    # HTML special chars
    HTML_CHARS = re.compile(r'[<>&"\']')
    HTML_ESCAPE = {'<': '&lt;', '>': '&gt;', '&': '&amp;', '"': '&quot;', "'": '&#x27;'}

    @classmethod
    def check_sql_injection(cls, value: str) -> bool:
        return bool(cls.SQL_INJECT.search(value))

    @classmethod
    def check_xss(cls, value: str) -> bool:
        return bool(cls.XSS_PATTERNS.search(value))

    @classmethod
    def check_path_traversal(cls, value: str) -> bool:
        return bool(cls.PATH_TRAVERSAL.search(value))

    @classmethod
    def html_escape(cls, value: str) -> str:
        return cls.HTML_CHARS.sub(lambda m: cls.HTML_ESCAPE[m.group()], value)

    @classmethod
    def sanitize_field(cls, name: str, value: Any) -> tuple:
        if not isinstance(value, str):
            return value, None

        # Check for threats
        if cls.check_sql_injection(value):
            return None, f"Field '{name}': potential SQL injection detected"
        if cls.check_xss(value):
            return None, f"Field '{name}': potential XSS detected"
        if name in ('file', 'path', 'filename') and cls.check_path_traversal(value):
            return None, f"Field '{name}': path traversal detected"

        # Sanitize
        sanitized = cls.html_escape(value.strip())
        return sanitized, None

    @classmethod
    def sanitize_request(cls, data: Dict[str, Any]) -> tuple:
        """Returns (sanitized_data, errors)"""
        sanitized = {}
        errors = {}

        for key, value in data.items():
            clean, error = cls.sanitize_field(key, value)
            if error:
                errors[key] = error
            else:
                sanitized[key] = clean

        return sanitized, errors


# Test cases
requests = [
    {
        'username': 'alice',
        'comment': 'Great tutorial! <b>Thanks</b>',
        'search': 'python regex',
    },
    {
        'username': "admin' OR '1'='1",
        'search': 'SELECT * FROM users',
    },
    {
        'username': 'bob',
        'comment': '<script>alert("XSS")</script>',
        'file': '../../../etc/passwd',
    },
]

print(f"\n   Input Sanitization Tests:")
for i, req in enumerate(requests, 1):
    print(f"\n   Request {i}: {req}")
    sanitized, errors = RequestSanitizer.sanitize_request(req)
    if errors:
        print(f"   → Blocked: {errors}")
    else:
        print(f"   → Sanitized: {sanitized}")
```

---

## 51.5 สรุป Part 51

```
URL Routing:
- Compile routes as regex patterns with named groups
- (?P<name>pattern) for URL parameters
- Match path against patterns in registration order
- Return groupdict() as kwargs to handler

Django path() converters:
<int:pk>   → (\d+)
<str:name> → ([^/]+)
<slug:s>   → ([-\w]+)
<uuid:id>  → ([0-9a-f]{8}-...)
<path:p>   → (.+)

FastAPI/Pydantic-style validation:
- Compile validator patterns once (module level)
- Use fullmatch() for field validation (not match/search)
- Collect all errors before raising (show all at once)
- Return validated + transformed values

Middleware sanitization:
- SQL injection: detect UNION, SELECT, --, etc.
- XSS: detect <script, javascript:, onerror=
- Path traversal: detect ../ or ..\\
- HTML escape: < > & " ' → HTML entities

Best practices:
✓ Compile patterns at module level (not per-request)
✓ Use fullmatch() for validation, search() for detection
✓ Return errors for all fields, not just first
✓ Distinguish "sanitize" (clean) vs "reject" (block)
✓ Log suspicious inputs for monitoring
✓ Test with known attack strings (OWASP test suite)
```

---

*[← Part 50: Lookahead & Lookbehind Advanced](part-50-lookahead-advanced.md) | [→ Part 52: Regex for Log Analysis & Monitoring](part-52-log-analysis.md)*
