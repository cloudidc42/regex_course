# Part 09: Lookahead และ Lookbehind

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-08

---

## 9.1 Lookaround คืออะไร?

**Lookaround** คือ assertion ที่ตรวจสอบ context รอบๆ ตำแหน่ง **โดยไม่ consume ตัวอักษร** (zero-width assertion)

```
Lookaround Types:
──────────────────────────────────────────────────────────
(?=pattern)    Positive Lookahead  — ตามด้วย pattern
(?!pattern)    Negative Lookahead  — ไม่ตามด้วย pattern
(?<=pattern)   Positive Lookbehind — นำหน้าด้วย pattern
(?<!pattern)   Negative Lookbehind — ไม่นำหน้าด้วย pattern
```

---

## 9.2 Positive Lookahead `(?=...)`

```python
import re

# จับตัวเลขที่ตามด้วย " dollars"
text = "I have 100 dollars and 50 euros"
amounts = re.findall(r'\d+(?=\s+dollars)', text)
print(amounts)  # ['100']

# จับ word ก่อน :
text2 = "name: Alice, age: 30, city: Bangkok"
keys = re.findall(r'\w+(?=:)', text2)
print(keys)  # ['name', 'age', 'city']
```

---

## 9.3 Negative Lookahead `(?!...)`

```python
import re

# จับ "Java" แต่ไม่ใช่ "JavaScript"
text = "Java and JavaScript and JavaEE"
java_only = re.findall(r'\bJava(?!Script|EE)\b', text)
print(java_only)  # ['Java']

# จับ file ที่ไม่ใช่ .log
files = "data.csv, error.log, info.log, report.pdf"
non_log = re.findall(r'\b\w+\.(?!log)\w+', files)
print(non_log)  # ['data.csv', 'report.pdf']
```

---

## 9.4 Positive Lookbehind `(?<=...)`

```python
import re

# จับตัวเลขที่ตามหลัง "$"
text = "Pay $100 or €50 or £30"
usd_amounts = re.findall(r'(?<=\$)\d+', text)
print(usd_amounts)  # ['100']

# จับ domain หลัง @
emails = "user@example.com, admin@company.org"
domains = re.findall(r'(?<=@)[\w.]+', emails)
print(domains)  # ['example.com', 'company.org']
```

---

## 9.5 Negative Lookbehind `(?<!...)`

```python
import re

# จับตัวเลขที่ไม่ได้ตามหลัง "$"
text = "I have $100 and 200 items at $50 each"
non_dollar = re.findall(r'(?<!\$)\b\d+\b', text)
print(non_dollar)  # ['200']

# จับ .js ที่ไม่ใช่ .min.js
files = "app.js, app.min.js, main.js, vendor.min.js"
non_min = re.findall(r'\b\w+(?<!\.min)\.js\b', files)
print(non_min)  # ['app.js', 'main.js']
```

---

## 9.6 Password Validation ขั้นสูง

```python
import re

def validate_password_advanced(password: str) -> dict:
    checks = [
        (r'(?=.{8,})', "อย่างน้อย 8 ตัวอักษร"),
        (r'(?=.*[A-Z])', "มี uppercase อย่างน้อย 1 ตัว"),
        (r'(?=.*[a-z])', "มี lowercase อย่างน้อย 1 ตัว"),
        (r'(?=.*\d)', "มีตัวเลขอย่างน้อย 1 ตัว"),
        (r'(?=.*[!@#$%^&*()_+\-=\[\]{}|;:,.<>?])', "มี special character"),
        (r'(?!.*(.){3,})', "ไม่มีตัวซ้ำกัน 3 ครั้งขึ้นไป"),
    ]
    
    results = {}
    for pattern, description in checks:
        results[description] = bool(re.match(pattern, password, re.IGNORECASE))
    
    passed = sum(results.values())
    strength_levels = ['Very Weak', 'Weak', 'Fair', 'Fair', 'Good', 'Strong', 'Very Strong']
    
    return {
        'score': f"{passed}/{len(results)}",
        'strength': strength_levels[min(passed, len(strength_levels)-1)],
        'checks': results,
        'valid': passed >= 5
    }

passwords = ['abc', 'Password1', 'P@ssw0rd!', 'Abcd1234!']
for pwd in passwords:
    result = validate_password_advanced(pwd)
    print(f"'{pwd}': {result['strength']} ({result['score']})")
```

---

## 9.7 Number Formatting

```python
import re

def format_number(n: str) -> str:
    """เพิ่ม comma separator ใน numbers"""
    return re.sub(r'(?<=\d)(?=(?:\d{3})+(?!\d))', ',', n)

numbers = ['1000', '1000000', '1234567890', '100']
for n in numbers:
    print(f"{n:15} → {format_number(n)}")

# ใส่ space หลัง punctuation
messy = "Hello,World!How are you?"
fixed = re.sub(r'([,!?])(?!\s)', r'\1 ', messy)
print(fixed)  # Hello, World! How are you?
```

---

## 9.8 สรุป Part 09

```
Syntax         ชื่อ                    ความหมาย
──────────────────────────────────────────────────────────
(?=pattern)    Positive Lookahead     ตามด้วย pattern
(?!pattern)    Negative Lookahead     ไม่ตามด้วย pattern
(?<=pattern)   Positive Lookbehind    นำหน้าด้วย pattern
(?<!pattern)   Negative Lookbehind    ไม่นำหน้าด้วย pattern
```

✅ Lookaround = zero-width assertion (ไม่ consume ตัวอักษร)  
✅ Lookbehind ต้องมีความยาวคงที่ใน Python `re`  
✅ ใช้ `regex` module สำหรับ variable-width lookbehind  

*[⬅ Part 08: Escape Sequences](part-08-escape-sequences.md) | [➡ Part 10: Flags](part-10-flags.md)*
