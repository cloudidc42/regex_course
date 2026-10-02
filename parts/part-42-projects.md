# Part 42: Real-world Projects — โปรเจกต์จริง

> **ระดับ:** สูง | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-41

---

## 42.1 Project: Config File Parser

```python
import re
from typing import Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class ConfigValue:
    value: Any
    type: str
    line: int
    comment: Optional[str] = None


class ConfigParser:
    """Parser สำหรับ .ini/.env/.conf files"""
    
    SECTION  = re.compile(r'^\[(?P<name>[^\]]+)\]\s*(?:;\s*(?P<comment>.+))?$')
    KV_PAIR  = re.compile(
        r'^(?P<key>[A-Za-z_][A-Za-z0-9_.-]*)\s*[=:]\s*'
        r'(?P<value>(?:"[^"]*"|\'[^\']*\'|[^#;\n]*))'
        r'\s*(?:[#;]\s*(?P<comment>.+))?$'
    )
    ENV_PAIR = re.compile(
        r'^(?:export\s+)?(?P<key>[A-Z_][A-Z0-9_]*)=(?P<value>.*)$'
    )
    COMMENT  = re.compile(r'^\s*[#;]')
    BLANK    = re.compile(r'^\s*$')
    
    INT_VAL   = re.compile(r'^-?\d+$')
    FLOAT_VAL = re.compile(r'^-?\d+\.\d+$')
    BOOL_VAL  = re.compile(r'^(?:true|false|yes|no|on|off)$', re.IGNORECASE)
    LIST_VAL  = re.compile(r'^(?:\w+,)+\w+$')
    
    @classmethod
    def _parse_value(cls, raw: str) -> tuple:
        raw = raw.strip()
        if (raw.startswith('"') and raw.endswith('"')) or \
           (raw.startswith("'") and raw.endswith("'")):
            return raw[1:-1], 'string'
        if cls.INT_VAL.match(raw):   return int(raw), 'int'
        if cls.FLOAT_VAL.match(raw): return float(raw), 'float'
        if cls.BOOL_VAL.match(raw):  return raw.lower() in ('true', 'yes', 'on'), 'bool'
        if cls.LIST_VAL.match(raw):  return [v.strip() for v in raw.split(',')], 'list'
        return raw, 'string'
    
    @classmethod
    def parse_ini(cls, text: str) -> Dict[str, Dict]:
        result = {'DEFAULT': {}}
        current_section = 'DEFAULT'
        
        for line_no, line in enumerate(text.splitlines(), 1):
            if cls.BLANK.match(line) or cls.COMMENT.match(line):
                continue
            section_m = cls.SECTION.match(line)
            if section_m:
                current_section = section_m.group('name').strip()
                result[current_section] = {}
                continue
            kv_m = cls.KV_PAIR.match(line)
            if kv_m:
                key = kv_m.group('key').strip()
                val, vtype = cls._parse_value(kv_m.group('value').strip())
                result[current_section][key] = ConfigValue(
                    value=val, type=vtype, line=line_no,
                    comment=kv_m.group('comment'),
                )
        return result
    
    @classmethod
    def parse_env(cls, text: str) -> Dict[str, str]:
        result = {}
        for line in text.splitlines():
            if cls.BLANK.match(line) or cls.COMMENT.match(line):
                continue
            m = cls.ENV_PAIR.match(line.strip())
            if m:
                key = m.group('key')
                value = m.group('value').strip()
                if len(value) >= 2 and value[0] in '"\'' and value[0] == value[-1]:
                    value = value[1:-1]
                result[key] = value
        return result


# ===== ทดสอบ =====
ini_config = """
; Application Configuration
[server]
host = localhost        ; server hostname
port = 8080
debug = true
workers = 4

[database]
host = db.example.com
port = 5432
name = myapp_db
pool_size = 10
ssl = false

[features]
enabled_modules = auth,logging,cache
api_version = 2
rate_limit = 1000.5
"""

env_config = """
# Environment Variables
APP_ENV=production
DATABASE_URL=postgresql://user:pass@localhost/db
SECRET_KEY="my-super-secret-key-2024"
DEBUG=false
PORT=8080
ALLOWED_HOSTS=localhost,example.com,api.example.com
export REDIS_URL='redis://localhost:6379/0'
"""

parser = ConfigParser

print("Config File Parser:")
print("=" * 60)

ini_result = parser.parse_ini(ini_config)
print("\n.ini parsing:")
for section, values in ini_result.items():
    if values:
        print(f"\n  [{section}]")
        for key, cv in values.items():
            print(f"    {key:<20} = {cv.value!r:<30} ({cv.type})")

print("\n.env parsing:")
env_result = parser.parse_env(env_config)
for key, value in env_result.items():
    masked = value if 'KEY' not in key and 'SECRET' not in key else '*' * len(value)
    print(f"  {key:<25} = {masked!r}")
```

---

## 42.2 Project: Template Engine

```python
import re
from typing import Dict, Any, Callable, List

class TemplateEngine:
    """Simple template engine ที่ใช้ regex"""
    
    VAR_TAG     = re.compile(r'\{\{\s*(?P<name>[\w.]+)\s*\}\}')
    IF_TAG      = re.compile(
        r'\{%\s*if\s+(?P<condition>[^%]+?)\s*%\}'
        r'(?P<body>.*?)'
        r'(?:\{%\s*else\s*%\}(?P<else_body>.*?))?'
        r'\{%\s*endif\s*%\}',
        re.DOTALL
    )
    FOR_TAG     = re.compile(
        r'\{%\s*for\s+(?P<var>\w+)\s+in\s+(?P<iterable>[\w.]+)\s*%\}'
        r'(?P<body>.*?)'
        r'\{%\s*endfor\s*%\}',
        re.DOTALL
    )
    COMMENT_TAG = re.compile(r'\{#.*?#\}', re.DOTALL)
    FILTER_TAG  = re.compile(r'\{\{\s*(?P<name>[\w.]+)\s*\|\s*(?P<filter>\w+)\s*\}\}')
    
    FILTERS: Dict[str, Callable] = {
        'upper':   str.upper,
        'lower':   str.lower,
        'title':   str.title,
        'strip':   str.strip,
        'length':  len,
        'reverse': lambda s: s[::-1],
    }
    
    def __init__(self):
        self.globals: Dict[str, Any] = {}
    
    def _resolve(self, name: str, context: Dict) -> Any:
        parts = name.split('.')
        obj = context.get(parts[0], self.globals.get(parts[0], ''))
        for part in parts[1:]:
            if isinstance(obj, dict):
                obj = obj.get(part, '')
            else:
                obj = getattr(obj, part, '')
        return obj
    
    def _eval_condition(self, condition: str, context: Dict) -> bool:
        condition = condition.strip()
        if condition.startswith('not '):
            return not self._eval_condition(condition[4:], context)
        for op, fn in [('==', lambda a, b: str(a) == str(b)),
                        ('!=', lambda a, b: str(a) != str(b))]:
            m = re.match(rf'^(?P<left>[\w.]+)\s*{re.escape(op)}\s*(?P<right>.+)$', condition)
            if m:
                left = self._resolve(m.group('left'), context)
                right = m.group('right').strip().strip('"\'')
                try:
                    return fn(left, right)
                except (ValueError, TypeError):
                    return False
        return bool(self._resolve(condition, context))
    
    def render(self, template: str, context: Dict) -> str:
        result = self.COMMENT_TAG.sub('', template)
        
        def render_for(m):
            var_name     = m.group('var')
            iterable_name = m.group('iterable')
            body         = m.group('body')
            items = self._resolve(iterable_name, context)
            if not items:
                return ''
            return ''.join(
                self.render(body, {**context, var_name: item})
                for item in items
            )
        result = self.FOR_TAG.sub(render_for, result)
        
        def render_if(m):
            condition = m.group('condition')
            body = m.group('body')
            else_body = m.group('else_body') or ''
            if self._eval_condition(condition, context):
                return self.render(body, context)
            return self.render(else_body, context)
        result = self.IF_TAG.sub(render_if, result)
        
        def render_filter(m):
            name = m.group('name')
            filter_name = m.group('filter')
            value = self._resolve(name, context)
            fn = self.FILTERS.get(filter_name, str)
            return str(fn(str(value)))
        result = self.FILTER_TAG.sub(render_filter, result)
        
        def render_var(m):
            value = self._resolve(m.group('name'), context)
            return str(value) if value is not None else ''
        return self.VAR_TAG.sub(render_var, result)


# ===== ทดสอบ =====
engine = TemplateEngine()

template = """
{# This is a comment - won't appear #}
Hello, {{ user.name | title }}!

{% if user.is_admin %}
  Welcome back, Administrator.
{% else %}
  You have {{ user.message_count }} messages.
{% endif %}

Your recent orders:
{% for order in orders %}
  - Order #{{ order.id }}: {{ order.product }} ({{ order.status | upper }})
{% endfor %}

{% if orders %}
Total orders: {{ order_count }}
{% endif %}
"""

context = {
    'user': {'name': 'alice smith', 'is_admin': False, 'message_count': 5},
    'orders': [
        {'id': '001', 'product': 'Python Book',  'status': 'delivered'},
        {'id': '002', 'product': 'Laptop Stand', 'status': 'processing'},
        {'id': '003', 'product': 'USB-C Hub',    'status': 'shipped'},
    ],
    'order_count': 3,
}

print("\nTemplate Engine:")
print("=" * 60)
rendered = engine.render(template, context)
print(rendered)
```

---

## 42.3 สรุป Part 42

```
Config Parser Patterns:
Section:   ^\[(?P<name>[^\]]+)\]
KV:        ^(?P<key>\w+)\s*[=:]\s*(?P<value>[^#\n]*)
Env:       ^(?:export\s+)?(?P<key>[A-Z_]\w*)=(?P<value>.*)
Comment:   ^\s*[#;]
Value types: INT r'^-?\d+$', FLOAT r'^-?\d+\.\d+$', BOOL r'^(true|false)$'

Template Engine Patterns:
Variable:  \{\{\s*(?P<name>[\w.]+)\s*\}\}
Filter:    \{\{\s*(?P<name>[\w.]+)\s*\|\s*(?P<filter>\w+)\s*\}\}
If:        \{%\s*if\s+(.+?)\s*%\}...\{%\s*endif\s*%\}
For:       \{%\s*for\s+\w+\s+in\s+\w+\s*%\}...\{%\s*endfor\s*%\}
Comment:   \{#.*?#\}

Processing order:
1. Comments (remove first)
2. Block tags (for, if)
3. Filtered variables
4. Plain variables last
```

---

*[← Part 41: Testing Regex](part-41-testing.md) | [→ Part 43: Regex Engine Internals](part-43-engine-internals.md)*
