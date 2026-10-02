# Part 37: Regex in Frameworks — Django, Flask, Express.js

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-36

---

## 37.1 Django URL Routing

```python
# Django ใช้ regex ใน urls.py (ก่อน Django 2.0 เป็นหลัก)
# และ re_path() ยังคงใช้ได้ใน Django 2.0+
import re
from typing import Dict, Optional, List, Callable, Any

# ===== Simulate Django URL routing =====
class UrlPattern:
    """จำลอง Django URL pattern"""
    
    def __init__(self, pattern: str, view: Callable, name: str = None):
        self.pattern = pattern
        self.view = view
        self.name = name
        self._regex = re.compile(f'^{pattern}$')
    
    def match(self, path: str) -> Optional[Dict]:
        m = self._regex.match(path)
        if m:
            return m.groupdict() or {}
        return None


class Router:
    """URL Router แบบ Django"""
    
    def __init__(self):
        self.patterns: List[UrlPattern] = []
    
    def path(self, pattern: str, view: Callable, name: str = None):
        """เพิ่ม URL pattern"""
        converted = self._django_to_regex(pattern)
        self.patterns.append(UrlPattern(converted, view, name))
    
    def re_path(self, pattern: str, view: Callable, name: str = None):
        """เพิ่ม raw regex pattern"""
        self.patterns.append(UrlPattern(pattern, view, name))
    
    def _django_to_regex(self, pattern: str) -> str:
        """แปลง Django path syntax เป็น regex
        <int:pk>  -> (?P<pk>[0-9]+)
        <str:slug>-> (?P<slug>[^/]+)
        <uuid:id> -> (?P<id>[0-9a-f-]{36})
        <path:p>  -> (?P<p>.+)
        """
        type_map = {
            'int':  r'[0-9]+',
            'str':  r'[^/]+',
            'slug': r'[-a-zA-Z0-9_]+',
            'uuid': r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}',
            'path': r'.+',
        }
        
        def replace_converter(m):
            converter = m.group(1)
            name = m.group(2)
            regex = type_map.get(converter, r'[^/]+')
            return f'(?P<{name}>{regex})'
        
        result = re.sub(r'<(\w+):(\w+)>', replace_converter, pattern)
        result = re.sub(r'<(\w+)>', r'(?P<\1>[^/]+)', result)
        return result
    
    def resolve(self, path: str) -> Optional[Dict]:
        """หา view สำหรับ path"""
        for pattern in self.patterns:
            kwargs = pattern.match(path)
            if kwargs is not None:
                return {'view': pattern.view, 'kwargs': kwargs, 'name': pattern.name}
        return None
    
    def reverse(self, name: str, **kwargs) -> Optional[str]:
        """สร้าง URL จาก named pattern"""
        for pattern in self.patterns:
            if pattern.name == name:
                url = pattern.pattern
                for key, value in kwargs.items():
                    url = re.sub(f'\\(\\?P<{key}>[^)]+\\)', str(value), url)
                url = url.replace('^', '').replace('$', '').replace('\\', '')
                return '/' + url
        return None


# ===== Views (mock) =====
def user_list(request, **kwargs): return f"User list"
def user_detail(request, **kwargs): return f"User {kwargs.get('pk')}"
def user_posts(request, **kwargs): return f"Posts of user {kwargs.get('pk')}"
def article_detail(request, **kwargs): return f"Article {kwargs.get('slug')}"
def category_view(request, **kwargs): return f"Category {kwargs.get('path')}"
def api_endpoint(request, **kwargs): return f"API version {kwargs.get('version')}"


# ===== URL Configuration =====
router = Router()

# Django path() syntax
router.path('users/',              user_list,   name='user-list')
router.path('users/<int:pk>/',     user_detail, name='user-detail')
router.path('users/<int:pk>/posts/',user_posts, name='user-posts')
router.path('articles/<slug:slug>/',article_detail, name='article-detail')
router.path('category/<path:path>/', category_view, name='category')

# Django re_path() — raw regex
router.re_path(
    r'api/v(?P<version>[12])/\w+/',
    api_endpoint,
    name='api'
)


# ===== ทดสอบ =====
test_paths = [
    '/users/',
    '/users/42/',
    '/users/99/posts/',
    '/articles/my-first-post/',
    '/category/tech/python/web/',
    '/api/v1/users/',
    '/api/v2/products/',
    '/nonexistent/',
]

print("Django URL Routing:")
print("=" * 60)
for path in test_paths:
    result = router.resolve(path.lstrip('/'))
    if result:
        view_name = result['view'].__name__
        print(f"  {path:<35} -> {view_name}({result['kwargs']})")
    else:
        print(f"  {path:<35} -> 404 Not Found")

print("\nURL Reverse:")
print(f"  user-detail pk=42  -> {router.reverse('user-detail', pk=42)}")
```

---

## 37.2 Flask Route Patterns

```python
import re
from typing import Dict, Optional, List, Callable, Tuple

class FlaskRouter:
    """จำลอง Flask URL routing"""
    
    # Flask converter types
    CONVERTERS = {
        'string': r'[^/]+',
        'int':    r'\\d+',
        'float':  r'\\d+\\.\\d+',
        'path':   r'.+',
        'uuid':   r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}',
        'any':    None,  # special
    }
    
    def __init__(self):
        self.routes: List[Tuple[re.Pattern, Callable, List[str]]] = []
        self._url_map: Dict[str, str] = {}
    
    def route(self, rule: str, methods: List[str] = None):
        """Decorator สำหรับ Flask route"""
        methods = methods or ['GET']
        def decorator(f):
            pattern, params = self._compile_rule(rule)
            self._url_map[f.__name__] = rule
            self.routes.append((pattern, f, methods))
            return f
        return decorator
    
    def _compile_rule(self, rule: str) -> Tuple[re.Pattern, List[str]]:
        """แปลง Flask rule เป็น regex pattern
        /users/<int:user_id>  ->  /users/(?P<user_id>\\d+)
        /files/<path:filename> -> /files/(?P<filename>.+)
        """
        params = []
        
        def replace(m):
            converter_type = m.group(1) or 'string'
            param_name = m.group(2)
            params.append(param_name)
            
            if converter_type == 'any':
                options = m.group(3).split(',') if m.group(3) else []
                regex = '|'.join(re.escape(o.strip()) for o in options)
                return f'(?P<{param_name}>{regex})'
            
            regex = self.CONVERTERS.get(converter_type, self.CONVERTERS['string'])
            return f'(?P<{param_name}>{regex})'
        
        # Pattern: <converter:name> or <name>
        pattern_str = re.sub(
            r'<(?:(\w+):)?(\w+)(?::([^>]*))?>', 
            replace, 
            re.escape(rule).replace('\\<', '<').replace('\\>', '>')
        )
        
        return re.compile(f'^{pattern_str}$'), params
    
    def match(self, path: str, method: str = 'GET') -> Optional[Dict]:
        for pattern, view, methods in self.routes:
            if method.upper() not in [m.upper() for m in methods]:
                continue
            m = pattern.match(path)
            if m:
                return {
                    'endpoint': view.__name__,
                    'view': view,
                    'params': m.groupdict(),
                }
        return None


# ===== Flask App Simulation =====
app = FlaskRouter()

@app.route('/api/users', methods=['GET'])
def get_users():
    return "List all users"

@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user():
    return "Get user"

@app.route('/api/users/<int:user_id>', methods=['PUT', 'PATCH'])
def update_user():
    return "Update user"

@app.route('/api/users/<int:user_id>', methods=['DELETE'])
def delete_user():
    return "Delete user"

@app.route('/files/<path:filename>', methods=['GET'])
def serve_file():
    return "Serve file"

@app.route('/api/v<int:version>/resource', methods=['GET'])
def versioned_api():
    return "Versioned API"


# ===== ทดสอบ =====
tests = [
    ('GET', '/api/users'),
    ('GET', '/api/users/42'),
    ('PUT', '/api/users/99'),
    ('DELETE', '/api/users/7'),
    ('GET', '/files/static/css/style.css'),
    ('GET', '/api/v2/resource'),
    ('POST', '/api/users'),  # method not allowed
]

print("\nFlask URL Routing:")
print("=" * 60)
for method, path in tests:
    result = app.match(path, method)
    if result:
        print(f"  {method:<7} {path:<40} -> {result['endpoint']}({result['params']})")
    else:
        print(f"  {method:<7} {path:<40} -> 405 Method Not Allowed / 404")
```

---

## 37.3 Express.js Middleware Pattern (Python Simulation)

```python
import re
from typing import List, Dict, Callable, Optional, Tuple, Any

class Request:
    def __init__(self, method: str, path: str, headers: Dict = None, body: Any = None):
        self.method = method
        self.path = path
        self.headers = headers or {}
        self.params: Dict = {}
        self.query: Dict = {}
        self.body = body

class Response:
    def __init__(self):
        self.status_code = 200
        self.headers: Dict = {}
        self.body = None
        self._sent = False
    
    def json(self, data): self.body = data; self._sent = True
    def status(self, code): self.status_code = code; return self
    def send(self, data): self.body = data; self._sent = True


class ExpressRouter:
    """Express.js-style Router in Python"""
    
    # Express path-to-regex: /users/:id  ->  /users/(?P<id>[^/]+)
    PARAM_RE = re.compile(r':(\w+)(?:\(([^)]+)\))?')
    WILDCARD = re.compile(r'\*')
    
    def __init__(self):
        self.middleware_stack: List[Tuple] = []
    
    def _compile_path(self, path: str) -> re.Pattern:
        """แปลง Express path เป็น regex
        /users/:id       -> /users/(?P<id>[^/]+)
        /files/*         -> /files/.*
        /api/v:ver(\\d+) -> /api/v(?P<ver>\\d+)
        """
        escaped = re.escape(path)
        escaped = escaped.replace('\\:', ':').replace('\\*', '*')
        
        def replace_param(m):
            name = m.group(1)
            constraint = m.group(2) or '[^/]+'
            return f'(?P<{name}>{constraint})'
        
        pattern = self.PARAM_RE.sub(replace_param, escaped)
        pattern = self.WILDCARD.sub('.*', pattern)
        return re.compile(f'^{pattern}(?:/)?$')
    
    def use(self, path_or_fn, fn: Callable = None):
        if callable(path_or_fn) and fn is None:
            self.middleware_stack.append((re.compile(r'^/'), 'ALL', path_or_fn))
        else:
            pattern = self._compile_path(path_or_fn)
            self.middleware_stack.append((pattern, 'ALL', fn))
    
    def _add_route(self, method: str, path: str, handler: Callable):
        pattern = self._compile_path(path)
        self.middleware_stack.append((pattern, method.upper(), handler))
    
    def get(self, path, handler):  self._add_route('GET', path, handler)
    def post(self, path, handler): self._add_route('POST', path, handler)
    def put(self, path, handler):  self._add_route('PUT', path, handler)
    def delete(self, path, handler): self._add_route('DELETE', path, handler)
    
    def handle(self, req: Request) -> Response:
        res = Response()
        idx = [0]
        
        def next_middleware():
            while idx[0] < len(self.middleware_stack):
                pattern, method, handler = self.middleware_stack[idx[0]]
                idx[0] += 1
                
                if method != 'ALL' and method != req.method.upper():
                    continue
                
                m = pattern.match(req.path)
                if m:
                    req.params.update(m.groupdict())
                    handler(req, res, next_middleware)
                    if res._sent:
                        return
        
        next_middleware()
        if not res._sent:
            res.status(404).send('Not Found')
        return res


# ===== Express App =====
router = ExpressRouter()

def logger(req, res, next):
    print(f"  [LOG] {req.method} {req.path}")
    next()

def require_auth(req, res, next):
    token = req.headers.get('Authorization', '')
    if not token.startswith('Bearer '):
        res.status(401).json({'error': 'Unauthorized'})
        return
    next()

def add_request_id(req, res, next):
    import hashlib, time
    req.headers['X-Request-ID'] = hashlib.md5(f"{time.time()}".encode()).hexdigest()[:8]
    next()

router.use(logger)
router.use(add_request_id)

def get_users(req, res, next):
    res.json({'users': [{'id': 1, 'name': 'Alice'}, {'id': 2, 'name': 'Bob'}]})

def get_user(req, res, next):
    user_id = req.params.get('id')
    res.json({'user': {'id': int(user_id), 'name': f'User {user_id}'}})

def create_user(req, res, next):
    res.status(201).json({'created': True, 'body': req.body})

router.get('/api/users', get_users)
router.get('/api/users/:id(\\d+)', get_user)

router.use('/api/admin', require_auth)
router.get('/api/admin/stats', lambda req, res, next: res.json({'uptime': 99.9}))

router.post('/api/users', create_user)


# ===== ทดสอบ =====
tests = [
    Request('GET',    '/api/users'),
    Request('GET',    '/api/users/42'),
    Request('GET',    '/api/admin/stats'),
    Request('GET',    '/api/admin/stats', {'Authorization': 'Bearer secret123'}),
    Request('POST',   '/api/users', body={'name': 'Charlie'}),
    Request('GET',    '/nonexistent'),
]

print("\nExpress.js Middleware:")
print("=" * 60)
for req in tests:
    res = router.handle(req)
    print(f"  -> {res.status_code} {str(res.body)[:50]}")
```

---

## 37.4 สรุป Part 37

```
Django URL Patterns:
path():     <int:pk>  -> (?P<pk>[0-9]+)
            <str:x>   -> (?P<x>[^/]+)
            <slug:x>  -> (?P<x>[-a-zA-Z0-9_]+)
            <path:x>  -> (?P<x>.+)
re_path():  raw regex, ใช้ named groups (?P<name>...)

Flask Routes:
<converter:name>  converter = string/int/float/path/uuid
<int:user_id>  -> (?P<user_id>\\d+)
<path:filename> -> (?P<filename>.+)

Express.js:
:param      -> (?P<param>[^/]+)
:id(\\d+)  -> (?P<id>\\d+)
*           -> .*
middleware chain: use() -> get/post/put/delete -> 404

Best practices:
- named groups > positional groups ใน web frameworks
- ลำดับ URL pattern สำคัญ (specific ก่อน general)
- middleware เรียงจาก global -> specific
- compile pattern ครั้งเดียวตอน startup ไม่ใช่ทุก request
```

---

*[← Part 36: Data Validation](part-36-data-validation.md) | [→ Part 38: Advanced Groups](part-38-advanced-groups.md)*
