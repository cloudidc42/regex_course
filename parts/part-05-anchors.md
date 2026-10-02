# Part 05: Anchors — ^, $, \b, \B

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-04

---

## 5.1 Anchors คืออะไร?

**Anchors** ไม่ match ตัวอักษร แต่ match **ตำแหน่ง** ในสตริง

```
Anchor  ความหมาย
──────────────────────────────────────────
^       จุดเริ่มต้นของสตริง (หรือบรรทัด)
$       จุดสิ้นสุดของสตริง (หรือบรรทัด)
\b      Word boundary (ขอบระหว่าง \w และ \W)
\B      Non-word boundary
\A      จุดเริ่มต้นของสตริง (Python)
\Z      จุดสิ้นสุดของสตริง (Python)
```

---

## 5.2 `^` และ `$`

```python
import re

text = "Hello World\nSecond Line\nThird Line"

# re.MULTILINE: ^ match จุดเริ่มต้นของแต่ละบรรทัด
all_starts = re.findall(r'^\w+', text, re.MULTILINE)
print(all_starts)  # ['Hello', 'Second', 'Third']

# ตรวจสอบ file extension
def get_extension(filename):
    m = re.search(r'\.([a-zA-Z0-9]+)$', filename)
    return m.group(1).lower() if m else None

files = ['document.pdf', 'image.JPEG', 'script.js', 'no_extension']
for f in files:
    print(f"  {f:25} → {get_extension(f)}")
```

---

## 5.3 `\b` — Word Boundary

```python
import re

text = "The cat catches many cats in that location"

# ❌ ไม่ใช้ \b
wrong = re.findall(r'cat', text)
print(wrong)  # ['cat', 'cat', 'cat', 'cat'] ← รวม catches, cats!

# ✅ ใช้ \b
correct = re.findall(r'\bcat\b', text)
print(correct)  # ['cat'] ← เฉพาะ "cat" คำเดียว
```

---

## 5.4 Validate Formats

```python
import re

validations = {
    'IPv4': r'^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$',
    'Hex Color': r'^#[0-9A-Fa-f]{6}$',
    'Date YYYY-MM-DD': r'^\d{4}-(?:0[1-9]|1[012])-(?:0[1-9]|[12]\d|3[01])$',
    'UUID': r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$',
}

test_values = {
    'IPv4': ['192.168.1.1', '256.0.0.1', '0.0.0.0'],
    'Hex Color': ['#FF5733', '#GGGGGG', '#FFF'],
    'Date YYYY-MM-DD': ['2026-10-02', '2026-13-01'],
    'UUID': ['550e8400-e29b-41d4-a716-446655440000', 'not-a-uuid'],
}

for name, pattern in validations.items():
    print(f"\n{name}:")
    for val in test_values[name]:
        valid = bool(re.fullmatch(pattern, val))
        print(f"  {'✅' if valid else '❌'} {val}")
```

---

## 5.5 ตัวอย่างจริง: INI Config Parser

```python
import re
from typing import Dict, Any

config_text = """
[database]
host = db.internal.company.com
port = 5432
name = production_db

[cache]
host = redis.internal.company.com
port = 6379
ttl = 3600
"""

class IniParser:
    SECTION_RE = re.compile(r'^\[([^\]]+)\]\s*$')
    KEY_RE = re.compile(r'^(\w+)\s*=\s*(.+?)\s*$')
    COMMENT_RE = re.compile(r'^\s*[;#]')
    EMPTY_RE = re.compile(r'^\s*$')
    
    @classmethod
    def parse(cls, text: str) -> Dict[str, Any]:
        result = {}
        current_section = None
        for line in text.splitlines():
            if cls.COMMENT_RE.match(line) or cls.EMPTY_RE.match(line):
                continue
            m = cls.SECTION_RE.match(line)
            if m:
                current_section = m.group(1)
                result[current_section] = {}
                continue
            m = cls.KEY_RE.match(line)
            if m and current_section:
                key, value = m.group(1), m.group(2)
                result[current_section][key] = value
        return result

config = IniParser.parse(config_text)
for section, values in config.items():
    print(f"\n[{section}]")
    for k, v in values.items():
        print(f"  {k} = {v}")
```

---

## 5.6 สรุป Part 05

✅ **`^`** — จุดเริ่มต้น  
✅ **`$`** — จุดสิ้นสุด  
✅ **`\b`** — Word boundary  
✅ **`\B`** — Non-word boundary  
✅ **`\A`** — Absolute start (Python)  
✅ **`\Z`** — Absolute end (Python)  
✅ **`re.MULTILINE`** — ให้ `^` และ `$` match แต่ละบรรทัด  
✅ **`re.fullmatch()`** — ต้องตรงทั้งสตริง  

*[⬅ Part 04: Quantifiers](part-04-quantifiers.md) | [➡ Part 06: Groups & Capturing](part-06-groups.md)*
