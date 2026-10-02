# Part 53: Regex Security Patterns — Detection & Defense

> **ระดับ:** สูง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-52

---

## 53.1 SQL Injection Detection

```python
import re
from typing import List, Dict, Tuple

print("SQL Injection Detection:")
print("=" * 60)

class SQLInjectionDetector:
    """Detect SQL injection attempts in user input"""

    PATTERNS = [
        (re.compile(r"'\s*(?:OR|AND)\s+'?\d", re.IGNORECASE),
         "Classic OR/AND injection"),
        (re.compile(r'\bUNION\s+(?:ALL\s+)?SELECT\b', re.IGNORECASE),
         "UNION SELECT injection"),
        (re.compile(r'--\s*$|/\*.*?\*/', re.MULTILINE),
         "SQL comment injection"),
        (re.compile(r';\s*(?:DROP|INSERT|UPDATE|DELETE|CREATE|ALTER)\b', re.IGNORECASE),
         "Stacked query injection"),
        (re.compile(r'\bAND\s+\d+=\d+', re.IGNORECASE),
         "Boolean blind injection"),
        (re.compile(r'\bSLEEP\s*\(|\bWAITFOR\s+DELAY\b|BENCHMARK\s*\(', re.IGNORECASE),
         "Time-based blind injection"),
        (re.compile(r'\bEXTRACTVALUE\s*\(|\bUPDATEXML\s*\(|\bGROUP\s+BY.*FLOOR', re.IGNORECASE),
         "Error-based injection"),
        (re.compile(r'\b(?:INFORMATION_SCHEMA|SYS\.TABLES|ALL_TABLES)\b', re.IGNORECASE),
         "Schema extraction attempt"),
        (re.compile(r'\b(?:XP_CMDSHELL|SP_EXECUTE|EXEC\s*\(|EXECUTE\s*\()\b', re.IGNORECASE),
         "System command execution"),
        (re.compile(r'\b(?:CHAR|CHR|NCHAR)\s*\(\d+', re.IGNORECASE),
         "Character encoding bypass"),
    ]

    @classmethod
    def scan(cls, value: str) -> List[Dict]:
        findings = []
        for pattern, description in cls.PATTERNS:
            m = pattern.search(value)
            if m:
                findings.append({
                    'type': description,
                    'match': m.group()[:50],
                    'position': m.start(),
                })
        return findings

    @classmethod
    def is_safe(cls, value: str) -> bool:
        return len(cls.scan(value)) == 0

    @classmethod
    def risk_score(cls, value: str) -> int:
        findings = cls.scan(value)
        score = 0
        weights = {
            'UNION SELECT': 10, 'Stacked query': 10, 'System command': 10,
            'Time-based': 8, 'Error-based': 8, 'Schema extraction': 7,
            'Classic OR/AND': 5, 'Boolean blind': 5,
            'SQL comment': 3, 'Character encoding': 3,
        }
        for f in findings:
            for key, w in weights.items():
                if key.lower() in f['type'].lower():
                    score += w
        return min(score, 100)


test_inputs = [
    ("normal input", "Hello World"),
    ("classic 1=1", "' OR '1'='1"),
    ("classic union", "1 UNION SELECT username,password FROM users--"),
    ("blind boolean", "1 AND 1=1"),
    ("time-based", "1; WAITFOR DELAY '0:0:5'--"),
    ("stacked", "1; DROP TABLE users; --"),
    ("schema enum", "1 UNION SELECT table_name FROM INFORMATION_SCHEMA.TABLES"),
    ("char bypass", "1 UNION SELECT CHAR(65,68,77,73,78)"),
    ("comment only", "normal -- this is a comment"),
]

print(f"\n   {'Input':<50} {'Safe':<8} {'Score'}")
print(f"   {'-'*50} {'-'*8} {'-'*6}")
for label, value in test_inputs:
    safe  = SQLInjectionDetector.is_safe(value)
    score = SQLInjectionDetector.risk_score(value)
    print(f"   {value[:48]:<50} {str(safe):<8} {score:3d}")
    if not safe:
        findings = SQLInjectionDetector.scan(value)
        for f in findings:
            print(f"     → {f['type']}: {f['match']!r}")
```

---

## 53.2 XSS Detection

```python
import re
from typing import List, Dict

print("\nXSS Detection:")
print("=" * 60)

class XSSDetector:
    """Detect Cross-Site Scripting attempts"""

    PATTERNS = [
        (re.compile(r'<\s*script[^>]*>', re.IGNORECASE),
         "Script tag"),
        (re.compile(r'\bon\w+\s*=\s*["\']?[^"\'>\s]+', re.IGNORECASE),
         "Event handler attribute"),
        (re.compile(r'javascript\s*:', re.IGNORECASE),
         "JavaScript protocol"),
        (re.compile(r'data\s*:[^,]*(?:script|javascript)', re.IGNORECASE),
         "Data URI with script"),
        (re.compile(r'(?:expression|eval)\s*\(', re.IGNORECASE),
         "Expression/eval injection"),
        (re.compile(r'vbscript\s*:', re.IGNORECASE),
         "VBScript protocol"),
        (re.compile(r'<\s*(?:iframe|object|embed|base|form|input)\s+[^>]*(?:src|action|href)\s*=', re.IGNORECASE),
         "Dangerous HTML element"),
        (re.compile(r'<\s*svg[^>]*>.*?</\s*svg\s*>', re.IGNORECASE | re.DOTALL),
         "SVG injection"),
        (re.compile(r'"\s*><\s*script|"\s*/>\s*<\s*script', re.IGNORECASE),
         "Attribute breakout + script"),
        (re.compile(r'&#x?[0-9a-f]+;.*?(?:script|javascript)', re.IGNORECASE),
         "Encoded XSS"),
    ]

    @classmethod
    def scan(cls, value: str) -> List[Dict]:
        findings = []
        for pattern, desc in cls.PATTERNS:
            for m in pattern.finditer(value):
                findings.append({'type': desc, 'match': m.group()[:60], 'start': m.start()})
        return findings

    @classmethod
    def sanitize(cls, value: str) -> str:
        HTML_ENTITIES = {
            '&': '&amp;', '<': '&lt;', '>': '&gt;',
            '"': '&quot;', "'": '&#x27;', '/': '&#x2F;',
        }
        return re.sub(r'[&<>"\'/]', lambda m: HTML_ENTITIES[m.group()], value)


xss_vectors = [
    "Hello World",
    "<script>alert('XSS')</script>",
    "<img src=x onerror=alert(1)>",
    "<a href='javascript:alert(1)'>click</a>",
    "<svg onload=alert(1)>",
    '"><script>alert(document.cookie)</script>',
    "<iframe src='javascript:alert(1)'>",
    "&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E;",
    "<img src=x oNeRrOr=alert(1)>",
]

print(f"\n   {'Input':<55} {'Safe'}")
print(f"   {'-'*55} {'-'*5}")
for vector in xss_vectors:
    findings = XSSDetector.scan(vector)
    safe = len(findings) == 0
    print(f"   {vector[:53]:<55} {safe}")
    if not safe:
        for f in findings[:1]:
            print(f"     → {f['type']}: {f['match'][:40]!r}")

print("\n   Sanitization:")
unsafe = '<script>alert("XSS")</script>'
print(f"   Input:  {unsafe!r}")
print(f"   Output: {XSSDetector.sanitize(unsafe)!r}")
```

---

## 53.3 Input Validation Patterns

```python
import re
from typing import Tuple

print("\nInput Validation Patterns:")
print("=" * 60)

VALIDATORS = {
    'email':        re.compile(r'^[a-zA-Z0-9._%+-]{1,64}@[a-zA-Z0-9.-]{1,253}\.[a-zA-Z]{2,}$'),
    'username':     re.compile(r'^[a-zA-Z][a-zA-Z0-9_-]{2,31}$'),
    'password':     re.compile(r'^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*]).{8,128}$'),
    'thai_id':      re.compile(r'^\d{13}$'),
    'passport':     re.compile(r'^[A-Z]{1,2}\d{6,9}$'),
    'phone_th':     re.compile(r'^0[6-9]\d{8}$'),
    'phone_intl':   re.compile(r'^\+?[1-9]\d{6,14}$'),
    'postcode_th':  re.compile(r'^\d{5}$'),
    'postcode_us':  re.compile(r'^\d{5}(?:-\d{4})?$'),
    'ip_v4':        re.compile(r'^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$'),
    'date_iso':     re.compile(r'^\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])$'),
    'time_24h':     re.compile(r'^(?:[01]\d|2[0-3]):[0-5]\d(?::[0-5]\d)?$'),
    'credit_card':  re.compile(r'^\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}$'),
    'amount':       re.compile(r'^\d{1,15}(?:\.\d{1,2})?$'),
    'url':          re.compile(r'^https?://[a-zA-Z0-9\-._~:/?#\[\]@!$&\'()*+,;=%]{1,2048}$'),
    'slug':         re.compile(r'^[a-z0-9]+(?:-[a-z0-9]+)*$'),
    'hex_color':    re.compile(r'^#(?:[0-9a-fA-F]{3}){1,2}$'),
}

def validate(field_type: str, value: str) -> Tuple[bool, str]:
    if field_type not in VALIDATORS:
        return False, f"Unknown field type: {field_type}"
    if VALIDATORS[field_type].fullmatch(value):
        return True, "OK"
    return False, f"Invalid {field_type}: {value!r}"


tests = [
    ('email',       'alice@example.com',    True),
    ('email',       'not-an-email',         False),
    ('phone_th',    '0891234567',           True),
    ('phone_th',    '1234567890',           False),
    ('thai_id',     '1234567890123',        True),
    ('thai_id',     '123456789012',         False),
    ('date_iso',    '2024-10-15',           True),
    ('date_iso',    '2024-13-01',           False),
    ('ip_v4',       '192.168.1.100',        True),
    ('ip_v4',       '999.0.0.1',            False),
    ('credit_card', '4111 1111 1111 1111',  True),
    ('url',         'https://example.com',  True),
    ('hex_color',   '#FF5733',              True),
    ('hex_color',   '#ZZZZZZ',              False),
    ('slug',        'my-great-post',        True),
    ('slug',        'Has Uppercase',        False),
]

print(f"\n   {'Type':<15} {'Value':<30} {'Expected':<10} {'Result'}")
print(f"   {'-'*15} {'-'*30} {'-'*10} {'-'*10}")
for field_type, value, expected in tests:
    valid, msg = validate(field_type, value)
    status = '✓ PASS' if valid == expected else '✗ FAIL'
    print(f"   {field_type:<15} {value:<30} {str(expected):<10} {status}")
```

---

## 53.4 สรุป Part 53

```
Security Regex Principles:

1. Detection vs Sanitization:
   Detection:    re.search() — find dangerous patterns
   Sanitization: re.sub()   — replace dangerous chars

2. SQL Injection patterns:
   OR/AND injection:   '\s*(?:OR|AND)\s+'?\d
   UNION-based:        \bUNION\s+(?:ALL\s+)?SELECT\b
   Stacked queries:    ;\s*(?:DROP|INSERT|...)
   Time-based:         SLEEP\(|WAITFOR DELAY
   Comment:            --\s*$|/\*.*?\*/

3. XSS patterns:
   Script tags:        <\s*script[^>]*>
   Event handlers:     \bon\w+\s*=
   JavaScript:         javascript\s*:
   HTML5 vectors:      <(iframe|object|embed|svg)[^>]*>

4. Input validation approach:
   - Use fullmatch() for validation
   - Validate at the boundary
   - Whitelist > blacklist
   - Specific patterns > generic
   - Bound lengths

5. Defense in depth:
   - Regex is the FIRST layer, not the only layer
   - Use prepared statements for SQL
   - Use templating engines for HTML
```

---

*[← Part 52: Log Analysis & Monitoring](part-52-log-analysis.md) | [→ Part 54: Regex Performance Optimization](part-54-performance.md)*
