# Part 50: Lookahead & Lookbehind — Advanced Techniques

> **ระดับ:** สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-49

---

## 50.1 Lookahead Patterns

```python
import re

print("Lookahead — Advanced:")
print("=" * 60)

# ===== Positive lookahead (?=...) =====
print("\n1. Positive Lookahead (?=pattern):")
print("   'match X only if followed by Y' — Y is NOT consumed")

# Match word followed by ':' but don't include ':'
text = "name: Alice, age: 30, city: Bangkok"
keys = re.findall(r'\w+(?=:)', text)
print(f"\n   Text: {text!r}")
print(f"   Words before ':': {keys}")

# ===== Negative lookahead (?!...) =====
print("\n2. Negative Lookahead (?!pattern):")
print("   'match X only if NOT followed by Y'")

# Match 'python' but not 'python3'
langs = "python python3 python2 python_code"
py_only = re.findall(r'python(?![23])', langs)
print(f"\n   Text: {langs!r}")
print(f"   'python' not followed by 2 or 3: {py_only}")

# Match numbers not followed by decimal
text = "price: 10.50, qty: 5, total: 100.00, count: 3"
integers = re.findall(r'\b\d+(?!\.\d)', text)
print(f"\n   Text: {text!r}")
print(f"   Integers (not decimals): {integers}")

# ===== Lookahead for validation =====
print("\n3. Password validation with lookaheads:")
# Strong password: 8+ chars, upper, lower, digit, special
PASSWORD = re.compile(
    r'^'
    r'(?=.*[A-Z])'          # at least one uppercase
    r'(?=.*[a-z])'          # at least one lowercase
    r'(?=.*\d)'             # at least one digit
    r'(?=.*[!@#$%^&*])'     # at least one special char
    r'.{8,}$'               # 8 or more chars total
)

passwords = [
    'password',       # no upper/digit/special
    'Password1',      # no special
    'P@ssword1',      # all requirements
    'Abcdefg!',       # no digit
    'Abc123!@',       # all requirements
    'short1!A',       # all requirements (8 chars)
    'short!A',        # no digit
]

print(f"\n   {'Password':<20} {'Valid':<8} {'Reason'}")
print(f"   {'-'*20} {'-'*8} {'-'*30}")
for pwd in passwords:
    valid = bool(PASSWORD.match(pwd))
    reasons = []
    if not re.search(r'[A-Z]', pwd):   reasons.append('no uppercase')
    if not re.search(r'[a-z]', pwd):   reasons.append('no lowercase')
    if not re.search(r'\d', pwd):       reasons.append('no digit')
    if not re.search(r'[!@#$%^&*]', pwd): reasons.append('no special')
    if len(pwd) < 8:                    reasons.append('too short')
    reason = ', '.join(reasons) if reasons else 'OK'
    print(f"   {pwd:<20} {str(valid):<8} {reason}")
```

---

## 50.2 Lookbehind Patterns

```python
import re

print("\nLookbehind — Advanced:")
print("=" * 60)

# ===== Positive lookbehind (?<=...) =====
print("\n1. Positive Lookbehind (?<=pattern):")
print("   'match X only if preceded by Y' — Y is NOT in result")

# Match currency amounts after '$'
text = "Price: $100, €50, ¥3000, £75"
usd_amounts = re.findall(r'(?<=\$)\d+', text)
print(f"\n   Text: {text!r}")
print(f"   After '$': {usd_amounts}")

# Match words after specific keywords
code = "import os, import sys, from pathlib import Path, from typing import Dict"
imported = re.findall(r'(?<=import )\w+', code)
print(f"\n   Code: {code!r}")
print(f"   Imported names: {imported}")

# ===== Negative lookbehind (?<!...) =====
print("\n2. Negative Lookbehind (?<!pattern):")
print("   'match X only if NOT preceded by Y'")

# Match digits not preceded by '$'
text = "I have $100 and €50 but only 30 in pocket"
non_currency = re.findall(r'(?<!\$)(?<!€)\b\d+\b', text)
print(f"\n   Text: {text!r}")
print(f"   Numbers not after currency: {non_currency}")

# Match 'log' not preceded by 'cat'
words = "dialog catalog log catalogue epilogue"
logs = re.findall(r'(?<!cat)(?<!a)log\b', words)
print(f"\n   Words: {words!r}")
print(f"   'log' not from 'catalog': {logs}")

# ===== Python limitation: fixed-length lookbehind =====
print("\n3. Python re: fixed-length lookbehind only:")
print("   Python re requires lookbehind to be fixed length!")
print("   Variable length → error!")
print()

# This works: (?<=ab) — fixed length 2
try:
    result = re.findall(r'(?<=ab)\w+', 'abcdef abxyz')
    print(f"   (?<=ab)\\w+: {result}  ← OK (fixed length)")
except re.error as e:
    print(f"   Error: {e}")

# This fails: (?<=a+) — variable length
try:
    result = re.findall(r'(?<=a+)\w+', 'abcdef')
    print(f"   (?<=a+)\\w+: {result}")
except re.error as e:
    print(f"   (?<=a+)\\w+: Error — {e}")  # Expected: error

# Workaround: use regex module for variable lookbehind
try:
    import regex
    result = regex.findall(r'(?<=a+)\w+', 'abcdef aaabcdef')
    print(f"\n   regex module (?<=a+)\\w+: {result}  ← OK with regex module")
except ImportError:
    print("\n   (regex module not available for variable lookbehind)")
```

---

## 50.3 Combined Lookahead + Lookbehind

```python
import re

print("\nCombined Lookahead + Lookbehind:")
print("=" * 60)

# ===== Extract between delimiters =====
print("\n1. Extract between delimiters (not including them):")
text = "[hello] [world] [foo bar]"

# Extract content between [ and ]
between = re.findall(r'(?<=\[)[^\]]*(?=\])', text)
print(f"   Text: {text!r}")
print(f"   Between []: {between}")

# Extract attributes in HTML
html = '<a href="link.html" class="btn" id="main">Click</a>'
attrs = re.findall(r'(?<=\s)\w+(?==)', html)
print(f"\n   HTML: {html!r}")
print(f"   Attribute names: {attrs}")

# ===== Boundary validation =====
print("\n2. Word boundary with lookahead+lookbehind:")

# Match exact word (stricter than \b for some cases)
text = "cat concatenate catfish scat"

# Using lookahead+lookbehind for word boundary
exact_cat = re.findall(r'(?<!\w)cat(?!\w)', text)
print(f"   Text: {text!r}")
print(f"   Exact 'cat': {exact_cat}")

# ===== Validate format with lookahead =====
print("\n3. Validate ISBN-13 structure:")
ISBN13 = re.compile(
    r'^'
    r'(?=\d{13}$)'           # exactly 13 digits total
    r'(?:978|979)'           # prefix
    r'-?\d{1,5}'             # group
    r'-?\d{1,7}'             # publisher
    r'-?\d{1,6}'             # item
    r'-?\d$'                 # check digit
)

isbns = [
    '978-0-13-468599-1',    # valid structure
    '9780134685991',         # valid (no hyphens)
    '978-3-16-148410-0',    # valid
    '979-1-234-56789-3',    # valid (979 prefix)
    '123-4-56-789012-3',    # invalid prefix
    '978-0-13-4685991',     # invalid (13 digits check fails)
]

print(f"\n   {'ISBN':<25} {'Valid'}")
print(f"   {'-'*25} {'-'*6}")
for isbn in isbns:
    valid = bool(ISBN13.match(isbn))
    print(f"   {isbn:<25} {valid}")

# ===== Lookahead to split without removing =====
print("\n4. Split on lookahead (split before uppercase):")
# Split camelCase into words
camel = "getCamelCaseWords"
# Split before each uppercase letter (except the first)
words = re.sub(r'(?<=[a-z])(?=[A-Z])', ' ', camel)
print(f"   {camel!r} → {words!r}")

camel_cases = [
    'camelCaseExample',
    'XMLParser',
    'getHTTPSConnection',
    'parseJSONData',
]
for c in camel_cases:
    # Handle consecutive uppercase (keep acronyms together)
    split = re.sub(r'(?<=[a-z])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])', ' ', c)
    print(f"   {c:<25} → {split!r}")
```

---

## 50.4 Lookaround for Data Extraction

```python
import re

print("\nLookaround for Data Extraction:")
print("=" * 60)

# ===== JSON field extraction =====
print("\n1. Extract JSON values without full parse:")
json_text = '''
{
    "name": "Alice Smith",
    "age": 30,
    "email": "alice@example.com",
    "active": true,
    "score": 98.5
}
'''

# Extract key-value pairs
kv = re.findall(r'"(\w+)":\s*(?:"([^"]*)"|([a-z]+|\d+(?:\.\d+)?))', json_text)
print("   Key-value pairs:")
for key, str_val, other_val in kv:
    val = str_val if str_val else other_val
    print(f"   {key}: {val!r}")

# ===== Log field extraction =====
print("\n2. Nginx access log parsing:")
NGINX_LOG = re.compile(
    r'(?P<ip>\d+\.\d+\.\d+\.\d+)'
    r'\s+-\s+-\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<method>\w+)\s+(?P<path>[^\s"]+)[^"]*"\s+'
    r'(?P<status>\d{3})\s+'
    r'(?P<bytes>\d+)'
)

log_lines = [
    '192.168.1.1 - - [15/Oct/2024:09:30:00 +0700] "GET /api/users HTTP/1.1" 200 1234',
    '10.0.0.2 - - [15/Oct/2024:09:30:01 +0700] "POST /api/login HTTP/1.1" 401 89',
    '172.16.0.5 - - [15/Oct/2024:09:30:02 +0700] "GET /static/style.css HTTP/1.1" 304 0',
]

print(f"\n   {'IP':<16} {'Status':<8} {'Method':<8} {'Path'}")
print(f"   {'-'*16} {'-'*8} {'-'*8} {'-'*30}")
for line in log_lines:
    m = NGINX_LOG.match(line)
    if m:
        d = m.groupdict()
        print(f"   {d['ip']:<16} {d['status']:<8} {d['method']:<8} {d['path']}")

# ===== Email header extraction =====
print("\n3. Email header parsing:")
headers_text = """From: Alice <alice@example.com>
To: Bob <bob@company.co.th>, Carol <carol@example.org>
Subject: Re: Project Update
Date: Mon, 15 Oct 2024 09:30:00 +0700
Message-ID: <msg123@example.com>
In-Reply-To: <msg100@example.com>
"""

# Extract header fields
HEADER = re.compile(r'^(?P<field>[A-Za-z-]+):\s+(?P<value>.+)$', re.MULTILINE)

headers = {}
for m in HEADER.finditer(headers_text):
    headers[m.group('field')] = m.group('value')

# Extract emails from headers
EMAIL_EXTRACT = re.compile(r'[\w.+-]+@[\w.-]+\.\w+')
print("\n   Headers:")
for field, value in headers.items():
    emails = EMAIL_EXTRACT.findall(value)
    if emails:
        print(f"   {field:<15}: {emails}")
    else:
        print(f"   {field:<15}: {value!r}")
```

---

## 50.5 สรุป Part 50

```
Lookahead:
(?=pattern)    positive: match if followed by pattern (not consumed)
(?!pattern)    negative: match if NOT followed by pattern

Lookbehind:
(?<=pattern)   positive: match if preceded by pattern (not consumed)
(?<!pattern)   negative: match if NOT preceded by pattern

Python re limitation:
- Lookbehind must be FIXED LENGTH
  (?<=\d{3}) ✓   (?<=\d+) ✗ (use regex module)

regex module (pip install regex):
- Variable-length lookbehind: (?<=\d+)
- Overlapping matches: regex.findall(pattern, text, overlapped=True)

Common patterns:
# Extract between delimiters (no capturing delimiters)
(?<=\[)[^\]]*(?=\])

# Words before/after specific punctuation
\w+(?=:)        before colon
(?<=@)\w+       after @

# Password strength
^(?=.*[A-Z])(?=.*\d)(?=.*[!@#]).{8,}$

# Split camelCase
(?<=[a-z])(?=[A-Z])

# Extract without including context
(?<=prefix)content(?=suffix)

Key insight: Lookaheads run at the SAME POSITION
Multiple (?=...) in sequence = AND conditions (all must pass)
Position advances ONLY after the main pattern, not lookarounds
```

---

*[← Part 49: Atomic Groups & Possessive Quantifiers](part-49-atomic-possessive.md) | [→ Part 51: Web Framework Integration](part-51-web-frameworks.md)*
