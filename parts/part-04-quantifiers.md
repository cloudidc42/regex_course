# Part 04: Quantifiers — *, +, ?, {n,m}

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-03

---

## 4.1 Quantifiers คืออะไร?

**Quantifier** คือตัวกำหนด **จำนวนครั้ง** ที่ element ก่อนหน้าต้องปรากฏ

```
Pattern  ความหมาย
─────────────────────────────────
*        0 ครั้งขึ้นไป (zero or more)
+        1 ครั้งขึ้นไป (one or more)
?        0 หรือ 1 ครั้ง (zero or one / optional)
{n}      n ครั้งพอดี (exactly n)
{n,}     n ครั้งขึ้นไป (n or more)
{n,m}    n ถึง m ครั้ง (between n and m)
```

---

## 4.2 `*` — Zero or More

```python
import re

# Optional whitespace
text = "key=value and key = value"
matches = re.findall(r'key\s*=\s*\S+', text)
print(matches)  # ['key=value', 'key = value']
```

---

## 4.3 `+` — One or More

```python
import re

text = "abc 123 def 456"
numbers = re.findall(r'\d+', text)
print(numbers)  # ['123', '456']
```

---

## 4.4 `?` — Optional

```python
import re

# colour หรือ color
text = "American color vs British colour"
matches = re.findall(r'colou?r', text)
print(matches)  # ['color', 'colour']

# Optional http/https
urls = re.findall(r'https?://\S+', 
                   "Visit http://example.com or https://secure.com")
print(urls)  # ['http://example.com', 'https://secure.com']
```

---

## 4.5 Greedy vs Lazy

```python
import re

html = "<b>Bold</b> and <i>Italic</i>"

# Greedy: .+ จับให้ได้มากที่สุด
greedy = re.findall(r'<.+>', html)
print(f"Greedy: {greedy}")
# ['<b>Bold</b> and <i>Italic</i>']

# Lazy: .+? จับให้ได้น้อยที่สุด
lazy = re.findall(r'<.+?>', html)
print(f"Lazy: {lazy}")
# ['<b>', '</b>', '<i>', '</i>']

# ดึง string ใน quotes
text = '"Hello" and "World"'
lazy_q = re.findall(r'"(.+?)"', text)
print(f"Lazy: {lazy_q}")  # ['Hello', 'World']
```

---

## 4.6 ตัวอย่างจริง: FormValidator

```python
import re
from dataclasses import dataclass
from typing import Optional

@dataclass
class ValidationResult:
    valid: bool
    message: str
    formatted: Optional[str] = None

class FormValidator:
    
    @staticmethod
    def email(email: str) -> ValidationResult:
        email = email.strip().lower()
        pattern = r'^[a-zA-Z0-9._%+\-]{1,64}@[a-zA-Z0-9.\-]{1,253}\.[a-zA-Z]{2,10}$'
        if re.match(pattern, email):
            return ValidationResult(True, "Valid email", email)
        return ValidationResult(False, "Invalid email format")
    
    @staticmethod
    def thai_phone(phone: str) -> ValidationResult:
        digits = re.sub(r'[^\d]', '', phone)
        if re.match(r'^0[689]\d{8}$', digits):
            formatted = f"{digits[:3]}-{digits[3:6]}-{digits[6:]}"
            return ValidationResult(True, "Valid Thai mobile", formatted)
        return ValidationResult(False, "Invalid Thai phone number")
    
    @staticmethod  
    def password(pwd: str) -> ValidationResult:
        errors = []
        if len(pwd) < 8:
            errors.append("อย่างน้อย 8 ตัวอักษร")
        if not re.search(r'[A-Z]', pwd):
            errors.append("มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
        if not re.search(r'[a-z]', pwd):
            errors.append("มีตัวพิมพ์เล็กอย่างน้อย 1 ตัว")
        if not re.search(r'\d', pwd):
            errors.append("มีตัวเลขอย่างน้อย 1 ตัว")
        if not errors:
            return ValidationResult(True, "Password ปลอดภัย")
        return ValidationResult(False, f"ต้องแก้ไข: {', '.join(errors)}")

# ทดสอบ
validator = FormValidator()
tests_email = ['user@example.com', 'invalid@', 'test+tag@company.co.th']
for val in tests_email:
    result = validator.email(val)
    print(f"{'✅' if result.valid else '❌'} {val}: {result.message}")
```

---

## 4.7 แบบฝึกหัด Part 04

```python
import re

data = """
ITEM-001: Widget A       $9.99    qty:100
ITEM-002: Gadget B Pro   $149.99  qty:25

Timestamps:
2026-01-15T08:30:00Z
2026-10-02T14:45:30.123Z
"""

# เฉลย
items = re.findall(r'ITEM-\d{3}', data)
prices = re.findall(r'\$\d+\.\d{2}', data)
qtys = re.findall(r'qty:(\d+)', data)
timestamps = re.findall(r'\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?(?:Z|[+-]\d{2}:\d{2})', data)

print(f"Items: {items}")
print(f"Prices: {prices}")
print(f"Quantities: {qtys}")
print(f"Timestamps: {timestamps}")
```

---

## 4.8 สรุป Part 04

✅ **`*`** — 0 ครั้งขึ้นไป  
✅ **`+`** — 1 ครั้งขึ้นไป  
✅ **`?`** — 0 หรือ 1 ครั้ง (optional)  
✅ **`{n}`** — n ครั้งพอดี  
✅ **`{n,m}`** — n ถึง m ครั้ง  
✅ **Greedy** — จับมากที่สุด (default)  
✅ **Lazy** — เพิ่ม `?` = จับน้อยที่สุด  

*[⬅ Part 03: Character Classes](part-03-character-classes.md) | [➡ Part 05: Anchors](part-05-anchors.md)*
