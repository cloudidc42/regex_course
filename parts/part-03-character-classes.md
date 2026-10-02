# Part 03: Character Classes และ Sets

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-02

---

## 3.1 Character Classes คืออะไร?

**Character Class** คือการระบุกลุ่มตัวอักษรที่ยอมรับได้ในตำแหน่งนั้น โดยใช้เครื่องหมาย `[ ]`

```python
import re

text = "The cat sat on a mat"
matches = re.findall(r'[csm]at', text)
print(matches)  # ['cat', 'sat', 'mat']
```

---

## 3.2 Character Ranges

```python
import re

text = "a1b2c3D4E5"
all_alphanum = re.findall(r'[a-zA-Z0-9]', text)
print(all_alphanum)  # ['a', '1', 'b', '2', 'c', '3', 'D', '4', 'E', '5']
```

### Ranges ที่ใช้บ่อย

```
[0-9]       → ตัวเลข 0-9 (เหมือน \d)
[a-z]       → ตัวอักษรพิมพ์เล็ก
[A-Z]       → ตัวอักษรพิมพ์ใหญ่
[a-zA-Z]    → ตัวอักษร (ไม่มีตัวเลข)
[a-zA-Z0-9] → alphanumeric (เหมือน \w แต่ไม่มี _)
[A-Fa-f0-9] → hex digit (case insensitive)
```

---

## 3.3 Negated Character Class `[^...]`

```python
import re

text = "abc123def456"
non_digits = re.findall(r'[^\d]+', text)
print(non_digits)  # ['abc', 'def']
```

---

## 3.4 Unicode Properties `\p{...}`

```python
# Python ต้องใช้ module 'regex' (pip install regex)
import regex

# \p{L} = Unicode Letter (ทุกภาษา)
text = "Hello สวัสดี 你好 مرحبا"
letters = regex.findall(r'\p{L}+', text)
print(letters)  # ['Hello', 'สวัสดี', '你好', 'مرحبا']

# \p{Script=Thai} = Thai script
thai_text = "สวัสดี Hello ไทย"
thai_only = regex.findall(r'\p{Script=Thai}+', thai_text)
print(thai_only)  # ['สวัสดี', 'ไทย']
```

---

## 3.5 ตัวอย่างจริง: Input Validation

```python
import re

def validate_username(username):
    pattern = r'^[a-zA-Z0-9][a-zA-Z0-9_-]{2,19}$'
    if re.match(pattern, username):
        return True, "✅ Valid username"
    return False, "Invalid username"

def validate_card_number(number):
    clean = re.sub(r'[\s-]', '', number)
    if not re.match(r'^\d+$', clean):
        return None, "Not a number"
    patterns = {
        'Visa': r'^4[0-9]{12}(?:[0-9]{3})?$',
        'Mastercard': r'^5[1-5][0-9]{14}$',
        'Amex': r'^3[47][0-9]{13}$',
        'Discover': r'^6(?:011|5[0-9]{2})[0-9]{12}$',
    }
    for card_type, pattern in patterns.items():
        if re.match(pattern, clean):
            return card_type, f"Valid {card_type}"
    return None, f"Unknown/Invalid ({len(clean)} digits)"

# Password strength check
def check_password_strength(password):
    checks = {
        'length': len(password) >= 8,
        'uppercase': bool(re.search(r'[A-Z]', password)),
        'lowercase': bool(re.search(r'[a-z]', password)),
        'digit': bool(re.search(r'[0-9]', password)),
        'special': bool(re.search(r'[!@#$%^&*()_+\-=\[\]{}|;:,.<>?/\\`~"\']', password)),
    }
    score = sum(checks.values())
    strength = ['Very Weak', 'Weak', 'Fair', 'Good', 'Strong', 'Very Strong'][score]
    return checks, score, strength

tests = ['abc', 'abcdefgh', 'Abcdefgh', 'Abcdef1h', 'Abcdef1!', 'A!bCd3f9']
for pwd in tests:
    checks, score, strength = check_password_strength(pwd)
    print(f"{pwd:15} → {strength} ({score}/5)")
```

---

## 3.6 แบบฝึกหัด Part 03

```python
import re

texts = {
    'hex': "#FF5733 and #00BFFF and #invalid",
    'version': "v1.0, v2.3.1, v10.0.5, v1.0.0-beta",
}

# เฉลย
hex_colors = re.findall(r'#[0-9A-Fa-f]{6}|#[0-9A-Fa-f]{3}', texts['hex'])
print(hex_colors)  # ['#FF5733', '#00BFFF']

versions = re.findall(r'v\d+\.\d+(?:\.\d+)?(?:-\w+)?', texts['version'])
print(versions)  # ['v1.0', 'v2.3.1', 'v10.0.5', 'v1.0.0-beta']
```

---

## 3.7 สรุป Part 03

✅ **`[abc]`** — Character class: match a, b, หรือ c  
✅ **`[a-z]`** — Range: match ตัวอักษร a ถึง z  
✅ **`[^abc]`** — Negation: match ทุกอย่างยกเว้น a, b, c  
✅ **Unicode `\p{...}`** — สำหรับ multi-language regex  

*[⬅ Part 02: Basic Characters](part-02-basic-characters.md) | [➡ Part 04: Quantifiers](part-04-quantifiers.md)*
