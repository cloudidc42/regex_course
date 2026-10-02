# Part 21: Advanced Groups — กลุ่มขั้นสูง

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-20

---

## 21.1 ทบทวน Groups พื้นฐาน

```python
import re

# Capturing group ธรรมดา
m = re.search(r'(\d{4})-(\d{2})-(\d{2})', '2026-10-02')
print(m.groups())           # ('2026', '10', '02')
print(m.group(0))           # '2026-10-02'  (whole match)
print(m.group(1))           # '2026'
print(m.group(2))           # '10'

# Non-capturing group (?:...)
m = re.search(r'(?:\d{4})-(\d{2})-(\d{2})', '2026-10-02')
print(m.groups())           # ('10', '02')  ไม่มีปีอยู่ใน groups
```

---

## 21.2 Named Groups (?P<name>...)

```python
import re

# Named groups
DATE = re.compile(r'(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})')
m = DATE.search('วันที่ 2026-10-02 เวลา 14:30')

if m:
    print(m.group('year'))         # '2026'
    print(m.group('month'))        # '10'
    print(m.group('day'))          # '02'
    print(m.groupdict())           # {'year': '2026', 'month': '10', 'day': '02'}

# Named group ใน substitution
result = DATE.sub(r'\g<day>/\g<month>/\g<year>', 'วันที่ 2026-10-02')
print(result)   # 'วันที่ 02/10/2026'

# Named group ใน re.sub callback
def format_date(m):
    d = m.groupdict()
    return f"{d['day']} {'มกราคม|กุมภาพันธ์|มีนาคม|เมษายน|พฤษภาคม|มิถุนายน|กรกฎาคม|สิงหาคม|กันยายน|ตุลาคม|พฤศจิกายน|ธันวาคม'.split('|')[int(d['month'])-1]} {int(d['year'])+543}"

result = DATE.sub(format_date, 'วันที่ 2026-10-02')
print(result)   # 'วันที่ 02 ตุลาคม 2569'
```

---

## 21.3 Named Backreferences (?P=name)

```python
import re

# Backreference ด้วยชื่อ
# ค้นหา HTML tag ที่มี opening และ closing ตรงกัน
TAG_PAIR = re.compile(r'<(?P<tag>\w+)[^>]*>.*?</(?P=tag)>', re.DOTALL | re.IGNORECASE)

html = """
<div class="main">content</div>
<span>text</span>
<p>paragraph</p>
<div>another</div>
"""

for m in TAG_PAIR.finditer(html):
    print(f"Tag: {m.group('tag'):10} | Content: {m.group(0)[:40]}")

# ค้นหาคำซ้ำ (double word detection)
DOUBLE_WORD = re.compile(r'\b(?P<word>\w+)\s+(?P=word)\b', re.IGNORECASE)

texts = [
    "the the quick brown fox",
    "she said said she was happy",
    "no duplicates here",
    "It is is important",
]

print("\nDouble Word Detection:")
for text in texts:
    matches = DOUBLE_WORD.findall(text)
    if matches:
        print(f"  FOUND: {text!r}")
        print(f"    Duplicates: {matches}")
    else:
        print(f"  OK:    {text!r}")
```

---

## 21.4 Non-Capturing Groups (?:...)

```python
import re

# ใช้ (?:...) เมื่อต้องการ group แต่ไม่ต้องการ capture
pattern_bad  = re.compile(r'(https?|ftp)://([\w.-]+)(/.*)?')
pattern_good = re.compile(r'(?:https?|ftp)://([\w.-]+)(?:/.*)?' )

urls = ['https://example.com/path', 'http://api.test.com', 'ftp://files.org/data']
print("Non-capturing groups:")
for url in urls:
    m = pattern_good.search(url)
    if m:
        print(f"  Domain: {m.group(1)}")

# (?:...) กับ quantifiers
p = re.compile(r'(?:ab){3}')
print("\n(?:ab){3}:")
for test in ['ababab', 'abab', 'abababab']:
    m = p.search(test)
    print(f"  {test!r}: {'MATCH' if m else 'NO MATCH'}")
```

---

## 21.5 Atomic Groups (?>...) — Python 3.11+

```python
import re

try:
    p_atomic = re.compile(r'(?>a+)b')
    tests = ['aaab', 'aaa', 'ab', 'b']
    print("Atomic Groups (Python 3.11+):")
    for t in tests:
        m = p_atomic.search(t)
        print(f"  {t!r}: {'MATCH' if m else 'NO MATCH'}")
except re.error:
    print("Atomic groups require Python 3.11+")
    # Workaround
    p_workaround = re.compile(r'(?=(a+))\1b')
    print("\nAtomic Group Workaround:")
    for t in ['aaab', 'aaa', 'ab', 'b']:
        m = p_workaround.search(t)
        print(f"  {t!r}: {'MATCH' if m else 'NO MATCH'}")
```

---

## 21.6 Conditional Groups (?(condition)yes|no)

```python
import re

# Conditional: match different patterns based on whether a group was captured
PRICE = re.compile(
    r'(?P<dollar>\$)?'
    r'(?(dollar)'
    r'\d{1,3}(?:,\d{3})*'
    r'|'
    r'\d{1,3}(?:\.\d{3})*'
    r')'
    r'(?P<cents>(?:\.|,)\d{2})?'
)

tests = ['$1,234.56', '$999', '1.234,56', '999']
print("Conditional Groups:")
for t in tests:
    m = PRICE.match(t)
    if m:
        currency = 'USD' if m.group('dollar') else 'EUR'
        print(f"  {t:15} -> {currency} {m.group(0)}")
    else:
        print(f"  {t:15} -> NO MATCH")

# Optional opening paren
PAREN_OPTIONAL = re.compile(r'(\()?+(\d+)(?(1)\))')
print("\nOptional parentheses:")
for t in ['(123)', '123', '(456']:
    m = PAREN_OPTIONAL.match(t)
    print(f"  {t!r}: {'MATCH: ' + m.group(0) if m else 'NO MATCH'}")
```

---

## 21.7 Named Groups สำหรับ Complex Parsing

```python
import re
from typing import Optional

class LogParser:
    """Parser ที่ใช้ named groups อย่างมีประสิทธิภาพ"""
    
    HTTP_REQUEST = re.compile(
        r'(?P<method>GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS)'
        r'\s+(?P<path>/[^\s]*)'
        r'(?:\?(?P<query>[^\s#]*))?'
        r'(?:#(?P<fragment>[^\s]*))?'
        r'\s+HTTP/(?P<version>[\d.]+)'
    )
    
    HTTP_HEADER = re.compile(
        r'^(?P<name>[A-Za-z][A-Za-z0-9-]*):\s*(?P<value>.+)$',
        re.MULTILINE
    )
    
    @classmethod
    def parse_request_line(cls, line: str) -> Optional[dict]:
        m = cls.HTTP_REQUEST.match(line)
        if not m:
            return None
        return {
            k: v for k, v in m.groupdict().items() if v is not None
        }
    
    @classmethod
    def parse_headers(cls, text: str) -> dict:
        return {
            m.group('name').lower(): m.group('value').strip()
            for m in cls.HTTP_HEADER.finditer(text)
        }


requests = [
    "GET /api/users?page=1&limit=10 HTTP/1.1",
    "POST /api/login HTTP/2.0",
    "DELETE /api/users/123 HTTP/1.1",
]

print("HTTP Request Parsing:")
for req in requests:
    parsed = LogParser.parse_request_line(req)
    if parsed:
        print(f"\n  Request: {req}")
        for k, v in parsed.items():
            print(f"    {k}: {v!r}")

header_text = """
Content-Type: application/json
Authorization: Bearer token123
X-Request-ID: req-abc-456
"""
headers = LogParser.parse_headers(header_text)
print("\nHeaders:")
for k, v in headers.items():
    print(f"  {k}: {v}")
```

---

## 21.8 สรุป Part 21

```
Group Types:
(...)           Capturing group — จับค่า, ใช้ \1, group(1)
(?:...)         Non-capturing — จัดกลุ่มแต่ไม่จับ
(?P<name>...)   Named group — เรียกด้วย group('name')
(?P=name)       Named backreference
(?>...)         Atomic group — Python 3.11+
(?(id)yes|no)   Conditional group

Performance Tips:
- ใช้ (?:...) แทน (...) เมื่อไม่จำเป็น
- Named groups อ่านง่ายกว่า แต่ช้ากว่าเล็กน้อย
- หลีกเลี่ยง nested quantified groups
```

---

*[← Part 20: Search & Replace](part-20-search-replace.md) | [→ Part 22: Lookahead & Lookbehind](part-22-lookahead-lookbehind.md)*
