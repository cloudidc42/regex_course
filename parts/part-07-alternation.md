# Part 07: Alternation — Pipe `|` และ OR Logic

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~45 นาที | **ข้อกำหนด:** Part 01-06

---

## 7.1 Alternation คืออะไร?

**Alternation** ใช้ `|` เพื่อระบุ "หรือ" (OR) ระหว่าง patterns

```python
import re

text = "I have a cat, a dog, and a bird"
animals = re.findall(r'cat|dog|bird', text)
print(animals)  # ['cat', 'dog', 'bird']

# ✅ วางที่ยาวกว่าก่อน
text2 = "I like Python and Java and JavaScript"
langs2 = re.findall(r'JavaScript|Java|Python', text2)
print(langs2)  # ['Python', 'JavaScript', 'Java']
```

---

## 7.2 Alternation Priority (Left-to-Right)

```python
import re

text = "JavaScript developer"

# ❌ Java จะถูกจับก่อน JavaScript
wrong = re.search(r'Java|JavaScript', text)
print(wrong.group())  # Java

# ✅ ใส่ที่ยาวกว่าก่อน
correct = re.search(r'JavaScript|Java', text)
print(correct.group())  # JavaScript
```

---

## 7.3 ตัวอย่างจริง: Multi-format Parser

```python
import re

# Parse เบอร์โทรศัพท์หลายรูปแบบ
phone_pattern = re.compile(
    r'(?:'
    r'\+\d{1,3}[-\s]?(?:\(\d+\)\s?)?\d{2,4}[-\s]?\d{3,4}[-\s]?\d{4}'
    r'|0[689]\d{1}[-\s]?\d{3}[-\s]?\d{4}'
    r'|0[2-8][-\s]?\d{3}[-\s]?\d{4}'
    r')'
)

phone_texts = """
Thai Mobile: 081-234-5678
Thai Landline: 02-123-4567
International: +66-81-234-5678
"""

for m in phone_pattern.finditer(phone_texts):
    print(f"Found: {m.group()}")

# File extensions
allowed = re.compile(
    r'\.(pdf|docx?|xlsx?'
    r'|jpe?g|png|gif|webp'
    r'|mp[34]|wav'
    r'|zip|tar\.gz'
    r'|html?|css|js|ts'
    r'|py|rb|go|java|php)$',
    re.IGNORECASE
)

files = ['document.pdf', 'image.jpg', 'script.js', 'video.mp4', 'unknown.xyz']
for f in files:
    m = allowed.search(f)
    status = '✅' if m else '❌'
    print(f"  {status} {f}")
```

---

## 7.4 สรุป Part 07

✅ **`a|b`** — a หรือ b  
✅ **Left-to-right priority** — ลอง alternative แรกก่อน  
✅ **ใส่ pattern ที่ยาวกว่าก่อน** — เพื่อป้องกัน partial match  
✅ **`(?:a|b)`** — group alternation โดยไม่ capture  

*[⬅ Part 06: Groups](part-06-groups.md) | [➡ Part 08: Escape Sequences](part-08-escape-sequences.md)*
