# Part 08: Escape Sequences

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~45 นาที | **ข้อกำหนด:** Part 01-07

---

## 8.1 Escape Sequences ทั้งหมด

```
Character    ความหมาย
───────────────────────────────────────────────
\\           literal backslash
\.           literal dot
\^           literal caret
\$           literal dollar
\*           literal asterisk
\+           literal plus
\?           literal question mark
\{           literal {
\}           literal }
\[           literal [
\]           literal ]
\(           literal (
\)           literal )
\|           literal pipe
\n           newline
\r           carriage return
\t           tab
\xNN         hex escape (NN = 2 hex digits)
\uNNNN       unicode escape (NNNN = 4 hex)
\UNNNNNNNN   unicode escape (8 hex, Python)
\b           word boundary (in regex), backspace (in strings)
```

---

## 8.2 Hex และ Unicode Escapes

```python
import re

text = "Café costs ₤ 5"  # Café costs ₤ 5

# Match é (U+00E9)
match = re.search(r'\xE9', text)
print(match.group() if match else "not found")  # é

# Unicode ranges สำหรับภาษาต่างๆ
thai_text = "ราคา 100 บาท"
thai_chars = re.findall(r'[฀-๿]+', thai_text)
print(thai_chars)  # ['ราคา', 'บาท']

# Arabic: U+0600-U+06FF
arabic = "السعر 100 دولار"
arabic_chars = re.findall(r'[؀-ۿ]+', arabic)
print(arabic_chars)

# CJK: U+4E00-U+9FFF
cjk = "价格 100 元"
cjk_chars = re.findall(r'[一-鿿]+', cjk)
print(cjk_chars)  # ['价格', '元']
```

---

## 8.3 re.escape() และ Context Matters

```python
import re

# นอก []: metacharacters ต้อง escape
text = "a-b.c+d*e?f"
outside = re.findall(r'\+|\*|\?|\.', text)
print(outside)  # ['.', '+', '*', '?']

# ใน []: ส่วนใหญ่ไม่ต้อง escape
inside = re.findall(r'[+*?.]', text)
print(inside)  # ['.', '+', '*', '?']

# re.escape()
special_chars = r'.^$*+?{}[]|()\'\"'
escaped = re.escape(special_chars)
print(escaped)
```

---

## 8.4 ตัวอย่างจริง: Search ใน Code

```python
import re

code = '''
def calculate_tax(price, rate=0.07):
    """Calculate tax: price * rate"""
    return price * rate

result = calculate_tax(100.00)
# formula: price * 0.07
'''

# ดึง function definitions
funcs = re.findall(r'def\s+(\w+)\s*\(', code)
print("Functions:", funcs)  # ['calculate_tax']

# ดึง parameters กับ default values
params = re.findall(r'(\w+)=([^,)]+)', code)
print("Params with defaults:", params)  # [('rate', '0.07')]

# ดึง float literals
floats = re.findall(r'\b\d+\.\d+\b', code)
print("Float literals:", floats)  # ['0.07', '100.00', '0.07']

# ดึง comments
comments = re.findall(r'#\s*(.+)$', code, re.MULTILINE)
print("Comments:", comments)
```

---

## 8.5 สรุป Part 08

✅ **`\.`** — literal dot  
✅ **`\n\t\r`** — whitespace characters  
✅ **`\xNN`** — hex escape  
✅ **`\uNNNN`** — unicode escape  
✅ **`re.escape()`** — escape string อัตโนมัติ  
✅ Context matters: บางอย่างต้อง escape นอก `[]` แต่ไม่ต้องใน `[]`  

*[⬅ Part 07: Alternation](part-07-alternation.md) | [➡ Part 09: Lookahead & Lookbehind](part-09-lookahead-lookbehind.md)*
