# Part 02: Basic Characters — ตัวอักษรพื้นฐานและการจับคู่

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01

---

## 2.1 Literal Characters (ตัวอักษรธรรมดา)

ตัวอักษรธรรมดาใน Regex จะ match กับตัวอักษรนั้นๆ ตรงๆ

```python
import re

# ตัวอักษรธรรมดาจะ match กับตัวมันเอง
text = "The quick brown fox"

# Match "fox"
match = re.search(r'fox', text)
print(match.group())  # fox

# Match "quick"
match = re.search(r'quick', text)
print(match.group())  # quick

# Case sensitive โดยค่าเริ่มต้น
match = re.search(r'Fox', text)  # F ตัวใหญ่
print(match)  # None (ไม่พบ!)

# ใช้ flag IGNORECASE
match = re.search(r'Fox', text, re.IGNORECASE)
print(match.group())  # fox
```

---

## 2.2 Metacharacters (ตัวอักษรพิเศษ)

ตัวอักษรเหล่านี้มีความหมายพิเศษใน Regex:

```
. ^ $ * + ? { } [ ] \ | ( )
```

```python
# ตัวอักษรพิเศษต้องใช้ backslash \ นำหน้าเพื่อ escape

# ตัวอย่าง: จับคู่ period (.)
text = "Price is $12.99 not $12x99"

# . (dot) ใน Regex หมายถึง "ตัวอักษรใดก็ได้"
matches = re.findall(r'12.99', text)
print(matches)  # ['12.99', '12x99'] ← จับ 12x99 ด้วย!

# ต้อง escape . เพื่อจับ literal period
matches = re.findall(r'12\.99', text)
print(matches)  # ['12.99'] ← ถูกต้อง!

# จับคู่ $ literal
text = "Cost: $100"
matches = re.findall(r'\$\d+', text)
print(matches)  # ['$100']

# จับคู่ ( และ ) literal
text = "Tel: (02) 123-4567"
matches = re.findall(r'\(\d+\)', text)
print(matches)  # ['(02)']
```

---

## 2.3 The Dot `.` — Wildcard

`.` (dot) match ตัวอักษรใดก็ได้ **ยกเว้น newline**

```python
import re

# . match ตัวอักษรใดก็ได้
text = "cat bat rat hat mat fat"
matches = re.findall(r'.at', text)
print(matches)  # ['cat', 'bat', 'rat', 'hat', 'mat', 'fat']

# . ไม่ match newline โดยค่าเริ่มต้น
text = "line1\nline2"
matches = re.findall(r'line.line', text)
print(matches)  # [] ← ไม่พบ!

# ใช้ re.DOTALL เพื่อให้ . match newline ด้วย
matches = re.findall(r'line.line', text, re.DOTALL)
print(matches)  # ['line1\nline2']
```

---

## 2.4 Character Classes: `\d`, `\w`, `\s`

```python
import re

# \d = [0-9] = ตัวเลข
text = "Order #12345 from customer ID: 67890"
digits = re.findall(r'\d+', text)
print(digits)  # ['12345', '67890']

# \w = [a-zA-Z0-9_]
text = "Hello_World-2026! user@test.com"
words = re.findall(r'\w+', text)
print(words)  # ['Hello_World', '2026', 'user', 'test', 'com']

# \s = whitespace
text = "Hello\tWorld\nFoo   Bar"
clean = re.sub(r'\s+', ' ', text)
print(clean)  # Hello World Foo Bar
```

---

## 2.5 Shorthand Class Summary

```
┌──────────────────────────────────────────────────────────┐
│              Character Class Shorthands                  │
├──────────┬────────────────────────┬───────────────────────┤
│ Class    │ Equivalent             │ ตัวอย่าง Match        │
├──────────┼────────────────────────┼───────────────────────┤
│ \d       │ [0-9]                  │ 0, 5, 9               │
│ \D       │ [^0-9]                 │ a, !, space           │
│ \w       │ [a-zA-Z0-9_]          │ a, Z, 5, _            │
│ \W       │ [^a-zA-Z0-9_]         │ !, @, space           │
│ \s       │ [ \t\n\r\f\v]         │ space, tab, newline   │
│ \S       │ [^ \t\n\r\f\v]        │ a, 1, !               │
│ \b       │ word boundary          │ ตำแหน่ง               │
│ \B       │ non-word boundary      │ ตำแหน่ง               │
│ \n       │ newline                │ \n                    │
│ \t       │ tab                    │ \t                    │
└──────────┴────────────────────────┴───────────────────────┘
```

---

## 2.6 ตัวอย่างจริง: Parse Access Log

```python
import re

access_log = """
192.168.1.100 - alice [02/Oct/2026:14:30:22 +0700] "GET /index.html HTTP/1.1" 200 1234
10.0.0.50 - bob [02/Oct/2026:14:31:05 +0700] "POST /api/login HTTP/1.1" 401 89
172.16.0.1 - - [02/Oct/2026:14:31:45 +0700] "GET /admin HTTP/1.1" 403 512
"""

log_pattern = (
    r'(?P<ip>\d{1,3}(?:\.\d{1,3}){3})'
    r' - '
    r'(?P<user>\S+)'
    r' \[(?P<datetime>[^\]]+)\]'
    r' "(?P<method>\w+) (?P<path>\S+) HTTP/[\d.]+"'
    r' (?P<status>\d{3})'
    r' (?P<bytes>\d+)'
)

entries = []
for line in access_log.strip().split('\n'):
    if not line:
        continue
    m = re.match(log_pattern, line)
    if m:
        entries.append(m.groupdict())

for entry in entries:
    print(f"IP: {entry['ip']:15} | Status: {entry['status']} | {entry['method']} {entry['path']}")
```

---

## 2.7 re.escape() — Escape String อัตโนมัติ

```python
import re

user_search = "price: $100.00 (special!)"
escaped = re.escape(user_search)
print(escaped)  # price:\ \$100\.00\ \(special!\)

text = "The price: $100.00 (special!) today"
matches = re.findall(escaped, text)
print(matches)  # ['price: $100.00 (special!)']
```

---

## 2.8 แบบฝึกหัด Part 02

```python
import re

text = """
File prices.txt has 1,250 records.
Cost: $49.99 per unit (minimum order: 10 units)
Tax rate: 7.5% applies to [electronics]
Email: admin+test@example.co.uk
Path: C:\\Users\\john\\Documents\\file.doc
"""

# เฉลย
prices = re.findall(r'\$\d+\.\d{2}', text)
percents = re.findall(r'\d+\.?\d*%', text)
in_brackets = re.findall(r'\[([^\]]+)\]', text)
win_paths = re.findall(r'[A-Z]:\\\\(?:\\w+\\\\)*\\w+\.\\w+', text)

print(f"Prices: {prices}")
print(f"Percents: {percents}")
print(f"In brackets: {in_brackets}")
```

---

## 2.9 สรุป Part 02

✅ **Literal Characters**: ตัวอักษรธรรมดา match กับตัวเอง  
✅ **Metacharacters**: `. ^ $ * + ? { } [ ] \ | ( )` มีความหมายพิเศษ  
✅ **Dot `.`**: match ตัวอักษรใดก็ได้ ยกเว้น newline  
✅ **`\d`**: digit, **`\w`**: word char, **`\s`**: whitespace  
✅ **`re.escape()`**: escape user input อัตโนมัติ  
✅ **Raw string `r"..."`**: ใช้เสมอสำหรับ Regex patterns  

*[⬅ Part 01: Introduction](part-01-introduction.md) | [➡ Part 03: Character Classes](part-03-character-classes.md)*
