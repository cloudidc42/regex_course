# Part 33: API Design Patterns — Regex สำหรับ REST API

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~65 นาที | **ข้อกำหนด:** Part 01-32

---

## 33.1 URL Routing Patterns

```python
import re
from typing import List, Dict, Optional, Callable, Any

class Router:
    """Simple regex-based URL router"""
    
    PARAM_TYPES = {
        'int':  r'(\d+)',
        'str':  r'([^/]+)',
        'slug': r'([a-z0-9-]+)',
        'uuid': r'([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})',
        'path': r'(.+)',
    }
    
    def __init__(self):
        self.routes: List[Dict] = []
    
    def _compile_route(self, template: str) -> re.Pattern:
        """แปลง route template เป็น regex
        
        /users/<int:id>  ->  ^/users/(\d+)$
        /posts/<slug>    ->  ^/posts/([a-z0-9-]+)$
        """
        param_re = re.compile(r'<(?:(?P<type>\w+):)?(?P<name>\w+)>')
        
        def replace_param(m: re.Match) -> str:
            ptype = m.group('type') or 'str'
            pattern = self.PARAM_TYPES.get(ptype, self.PARAM_TYPES['str'])
            return pattern
        
        regex_str = '^' + param_re.sub(replace_param, template) + '$'
        return re.compile(regex_str)
    
    def _extract_param_names(self, template: str) -> List[str]:
        param_re = re.compile(r'<(?:\w+:)?(\w+)>')
        return param_re.findall(template)
    
    def add_route(self, method: str, template: str, handler: Callable):
        pattern = self._compile_route(template)
        names = self._extract_param_names(template)
        self.routes.append({
            'method': method.upper(),
            'template': template,
            'pattern': pattern,
            'param_names': names,
            'handler': handler,
        })
    
    def match(self, method: str, path: str) -> Optional[Dict]:
        """Match URL path และ return handler + params"""
        for route in self.routes:
            if route['method'] != method.upper():
                continue
            m = route['pattern'].match(path)
            if m:
                params = dict(zip(route['param_names'], m.groups()))
                return {
                    'handler': route['handler'],
                    'params': params,
                    'template': route['template'],
                }
        return None


# ทดสอบ Router
router = Router()

router.add_route('GET', '/users', lambda: "list users")
router.add_route('GET', '/users/<int:id>', lambda id: f"user {id}")
router.add_route('GET', '/posts/<slug:slug>', lambda slug: f"post {slug}")
router.add_route('GET', '/files/<path:path>', lambda path: f"file {path}")
router.add_route('GET', '/items/<uuid:uuid>', lambda uuid: f"item {uuid}")

test_urls = [
    ('GET', '/users'),
    ('GET', '/users/42'),
    ('GET', '/posts/my-first-post'),
    ('GET', '/files/docs/api/v1.md'),
    ('GET', '/items/550e8400-e29b-41d4-a716-446655440000'),
    ('GET', '/unknown/path'),
    ('POST', '/users/42'),
]

print("URL Router Test:")
print("=" * 60)
for method, path in test_urls:
    result = router.match(method, path)
    if result:
        print(f"  OK {method:4} {path:<50} params={result['params']}")
    else:
        print(f"  XX {method:4} {path:<50} [no match]")
```

---

## 33.2 Query String Parser

```python
import re
from typing import Dict, List, Any
from urllib.parse import unquote_plus

class QueryStringParser:
    """Parse and validate query string parameters"""
    
    QUERY_PAIR = re.compile(r'([^=&]+)=([^&]*)')
    ARRAY_KEY  = re.compile(r'^([^\[]+)\[(\d*)\]$')
    NESTED_KEY = re.compile(r'^([^\[]+)\[([^\]]*)\](.*)$')
    
    VALIDATORS = {
        'int':    re.compile(r'^-?\d+$'),
        'uint':   re.compile(r'^\d+$'),
        'float':  re.compile(r'^-?\d+(?:\.\d+)?$'),
        'bool':   re.compile(r'^(?:true|false|1|0|yes|no)$', re.IGNORECASE),
        'date':   re.compile(r'^\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])$'),
        'email':  re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'),
        'slug':   re.compile(r'^[a-z0-9-]+$'),
        'uuid':   re.compile(r'^[0-9a-f]{8}-(?:[0-9a-f]{4}-){3}[0-9a-f]{12}$', re.IGNORECASE),
        'order':  re.compile(r'^(?:asc|desc)$', re.IGNORECASE),
    }
    
    @classmethod
    def parse(cls, query_string: str) -> Dict[str, Any]:
        result: Dict[str, Any] = {}
        query_string = query_string.lstrip('?')
        
        for m in cls.QUERY_PAIR.finditer(query_string):
            key = unquote_plus(m.group(1))
            value = unquote_plus(m.group(2))
            
            arr_m = cls.ARRAY_KEY.match(key)
            if arr_m:
                base = arr_m.group(1)
                if base not in result:
                    result[base] = []
                if isinstance(result[base], list):
                    result[base].append(value)
            else:
                nested_m = cls.NESTED_KEY.match(key)
                if nested_m:
                    parent = nested_m.group(1)
                    child = nested_m.group(2)
                    if parent not in result:
                        result[parent] = {}
                    if isinstance(result[parent], dict):
                        result[parent][child] = value
                else:
                    if key in result:
                        if not isinstance(result[key], list):
                            result[key] = [result[key]]
                        result[key].append(value)
                    else:
                        result[key] = value
        
        return result
    
    @classmethod
    def validate_param(cls, value: str, vtype: str) -> bool:
        validator = cls.VALIDATORS.get(vtype)
        if not validator:
            return True
        return bool(validator.match(str(value)))
    
    @classmethod
    def validate_schema(cls, params: Dict, schema: Dict[str, Dict]) -> Dict:
        errors = {}
        result = {}
        
        for field, rules in schema.items():
            value = params.get(field)
            
            if rules.get('required') and value is None:
                errors[field] = f"'{field}' is required"
                continue
            
            if value is None:
                result[field] = rules.get('default')
                continue
            
            vtype = rules.get('type', 'str')
            if not cls.validate_param(str(value), vtype):
                errors[field] = f"'{field}' must be of type {vtype}"
                continue
            
            if vtype in ('int', 'uint', 'float'):
                num = float(value)
                if 'min' in rules and num < rules['min']:
                    errors[field] = f"'{field}' must be >= {rules['min']}"
                    continue
                if 'max' in rules and num > rules['max']:
                    errors[field] = f"'{field}' must be <= {rules['max']}"
                    continue
            
            if 'enum' in rules and value not in rules['enum']:
                errors[field] = f"'{field}' must be one of: {rules['enum']}"
                continue
            
            result[field] = value
        
        return {'valid': not errors, 'errors': errors, 'data': result}


# ทดสอบ
test_queries = [
    "page=2&limit=20&sort=name&order=asc",
    "filter[status]=active&filter[role]=admin",
    "tag=python&tag=regex&tag=backend",
    "items[]=apple&items[]=banana&items[]=cherry",
]

parser = QueryStringParser
print("Query String Parser:")
print("=" * 60)
for qs in test_queries:
    parsed = parser.parse(qs)
    print(f"  Input:  {qs}")
    print(f"  Result: {parsed}\n")

schema = {
    'page':  {'type': 'uint', 'default': '1', 'min': 1, 'max': 1000},
    'limit': {'type': 'uint', 'default': '20', 'min': 1, 'max': 100},
    'order': {'type': 'order', 'default': 'asc', 'enum': ['asc', 'desc']},
}

test_params = [
    {'page': '3', 'limit': '10', 'order': 'desc'},
    {'page': '-1', 'limit': '200', 'order': 'INVALID'},
]

print("Schema Validation:")
for params in test_params:
    result = parser.validate_schema(params, schema)
    status = "VALID" if result['valid'] else "INVALID"
    print(f"  {params} -> {status}")
    if result['errors']:
        for err in result['errors'].values():
            print(f"    - {err}")
```

---

## 33.3 API Response Transformer

```python
import re
from typing import Any, Dict, List

class ResponseTransformer:
    """Transform and filter API responses with regex"""
    
    CAMEL_TO_SNAKE = re.compile(r'(?<=[a-z0-9])([A-Z])|(?<=[A-Z])([A-Z])(?=[a-z])')
    SNAKE_TO_CAMEL = re.compile(r'_([a-z])')
    
    CREDIT_CARD = re.compile(r'\b(\d{4})\d{8}(\d{4})\b')
    PHONE = re.compile(r'(\+?66|0)([6-9]\d)(\d{3})(\d{4})')
    EMAIL_MASK = re.compile(r'([a-zA-Z0-9._%+-]{2})[a-zA-Z0-9._%+-]+@([a-zA-Z0-9.-]+\.[a-zA-Z]{2,})')
    
    @classmethod
    def camel_to_snake(cls, name: str) -> str:
        name = cls.CAMEL_TO_SNAKE.sub(lambda m: '_' + (m.group(1) or m.group(2)), name)
        return name.lower()
    
    @classmethod
    def snake_to_camel(cls, name: str) -> str:
        return cls.SNAKE_TO_CAMEL.sub(lambda m: m.group(1).upper(), name)
    
    @classmethod
    def transform_keys(cls, data: Any, direction: str = 'to_camel') -> Any:
        transformer = cls.snake_to_camel if direction == 'to_camel' else cls.camel_to_snake
        if isinstance(data, dict):
            return {transformer(k): cls.transform_keys(v, direction) for k, v in data.items()}
        if isinstance(data, list):
            return [cls.transform_keys(item, direction) for item in data]
        return data
    
    @classmethod
    def mask_sensitive(cls, data: Any) -> Any:
        if isinstance(data, str):
            data = cls.CREDIT_CARD.sub(r'\1****\2', data)
            data = cls.PHONE.sub(r'\1\2***\4', data)
            data = cls.EMAIL_MASK.sub(r'\1***@\2', data)
            return data
        if isinstance(data, dict):
            result = {}
            for k, v in data.items():
                if re.search(r'password|secret|token|cvv|pin', k, re.IGNORECASE):
                    result[k] = '***'
                else:
                    result[k] = cls.mask_sensitive(v)
            return result
        if isinstance(data, list):
            return [cls.mask_sensitive(item) for item in data]
        return data
    
    @classmethod
    def filter_fields(cls, data: Dict, include: List[str] = None, exclude: List[str] = None) -> Dict:
        if include:
            include_pats = [re.compile('^' + p.replace('*', '.*') + '$') for p in include]
            data = {k: v for k, v in data.items() if any(p.match(k) for p in include_pats)}
        if exclude:
            exclude_pats = [re.compile('^' + p.replace('*', '.*') + '$') for p in exclude]
            data = {k: v for k, v in data.items() if not any(p.match(k) for p in exclude_pats)}
        return data


# ทดสอบ
transformer = ResponseTransformer

snake_data = {
    'user_id': 1,
    'first_name': 'Alice',
    'email_address': 'alice@example.com',
    'address_info': {'street_name': '123 Main St', 'zip_code': '10100'}
}

camel = transformer.transform_keys(snake_data, 'to_camel')
back  = transformer.transform_keys(camel, 'to_snake')

print("Key Transformation:")
print(f"  snake->camel: {camel}")
print(f"  camel->snake: {back}")

sensitive = {
    'name': 'Bob',
    'email': 'bob.smith@example.com',
    'phone': '0812345678',
    'credit_card': '4532015112830366',
    'password': 'mysecret123',
    'api_token': 'sk-abc123',
}

masked = transformer.mask_sensitive(sensitive)
print("\nSensitive Masking:")
for k, v in masked.items():
    print(f"  {k:15}: {v}")

print("\nField Filtering:")
filtered = transformer.filter_fields(sensitive, exclude=['password', 'api_*', 'credit_*'])
print(f"  exclude [password, api_*, credit_*]: {list(filtered.keys())}")
```

---

## 33.4 สรุป Part 33

```
API Routing (template -> regex):
/users/<int:id>    -> ^/users/(\d+)$
/posts/<slug>      -> ^/posts/([a-z0-9-]+)$
/files/<path:path> -> ^/files/(.+)$
/items/<uuid>      -> ^/items/([0-9a-f]{8}-...){12})$

Query String:
key=value          -> simple pair
key[]=v1&key[]=v2  -> array notation
key[sub]=value     -> nested object

Key Transformation:
camelCase->snake:  (?<=[a-z0-9])([A-Z]) -> _\1.lower()
snake->camelCase:  _([a-z]) -> \1.upper()

Sensitive Masking:
Credit card: (\d{4})\d{8}(\d{4}) -> \1****\2
Phone:       (\+?66|0)([6-9]\d)(\d{3})(\d{4}) -> \1\2***\4
Email:       ([a-zA-Z]{2})[a-zA-Z._%+-]+@ -> \1***@

Field filter wildcards: api_* -> ^api_.*$
```

---

*[← Part 32: Code Analysis](part-32-code-analysis.md) | [→ Part 34: Log Analysis](part-34-log-analysis.md)*
