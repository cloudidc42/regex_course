# Part 24: Unicode & International Text — ข้อความนานาชาติ

> **ระดับ:** กลาง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-23

---

## 24.1 Unicode Basics

```python
import re

# Python 3 strings are Unicode by default
texts = ["Hello World", "สวัสดี ไทย", "こんにちは 日本語", "مرحبا عربي"]
for text in texts:
    words = re.findall(r'\w+', text)
    print(f"  {text!r}: {words}")

# \w กับ UNICODE (default) = letters, digits, underscore ทุกภาษา
thai_digits = "๐๑๒๓๔๕๖๗๘๙"
found = re.findall(r'\d', thai_digits)
print(f"\nThai digits \\d: {found}")

# re.ASCII จำกัดเฉพาะ ASCII
words_ascii = re.findall(r'\w+', "สวัสดี World", re.ASCII)
print(f"Thai with re.ASCII: {words_ascii}")   # ['World'] only
```

---

## 24.2 Unicode Properties (regex module)

```python
try:
    import regex

    # \p{L} = Letter, \p{N} = Number, \p{Thai} = Thai
    texts = ["Hello สวัสดี 123 !@#", "Python ไพธอน"]
    print("Unicode Properties:")
    for text in texts:
        letters = regex.findall(r'\p{L}+', text)
        numbers = regex.findall(r'\p{N}+', text)
        thai    = regex.findall(r'\p{Thai}+', text)
        print(f"  {text!r}")
        print(f"    Letters: {letters}")
        if thai: print(f"    Thai: {thai}")
except ImportError:
    print("pip install regex  for \\p{L}, \\p{Thai} etc.")
```

---

## 24.3 Thai Text Patterns

```python
import re

THAI_CHAR  = r'[฀-๿]'
THAI_DIGIT = r'[๐-๙]'

# 1. Thai words
thai_text = "ประเทศไทยตั้งอยู่ในเอเชียตะวันออกเฉียงใต้ ประชากร 70 ล้านคน"
thai_words = re.findall(rf'{THAI_CHAR}+', thai_text)
print(f"Thai words: {thai_words[:5]}...")

# 2. Thai digit extraction
thai_nums = re.findall(rf'{THAI_DIGIT}+', "ราคา ๑,๒๕๐ บาท")
print(f"Thai numbers: {thai_nums}")

# 3. Thai <-> Arabic digit conversion
def thai_to_arabic(text: str) -> str:
    return text.translate(str.maketrans('๐๑๒๓๔๕๖๗๘๙', '0123456789'))

def arabic_to_thai(text: str) -> str:
    return text.translate(str.maketrans('0123456789', '๐๑๒๓๔๕๖๗๘๙'))

print(f"\n'๑๒๓๔๕' → '{thai_to_arabic('๑๒๓๔๕')}'")
print(f"'12345' → '{arabic_to_thai('12345')}'")

# 4. Thai ID card
THAI_ID = re.compile(r'\b\d-\d{4}-\d{5}-\d{2}-\d\b')
ids = ["3-1001-00123-45-6", "1234567890123", "3-1001-00123-45"]
for id_str in ids:
    status = 'valid' if THAI_ID.match(id_str) else 'invalid'
    print(f"  {id_str}: {status}")
```

---

## 24.4 Unicode Normalization

```python
import re
import unicodedata

# café อาจเขียนได้ 2 วิธี
cafe_nfc = "café"      # é as single codepoint
cafe_nfd = "café"     # e + combining accent

print(f"NFC: {cafe_nfc!r} len={len(cafe_nfc)}")
print(f"NFD: {cafe_nfd!r} len={len(cafe_nfd)}")
print(f"NFC == NFD: {cafe_nfc == cafe_nfd}")

# Normalize before matching
def normalize_text(text: str, form: str = 'NFC') -> str:
    return unicodedata.normalize(form, text)

cafe_normalized = normalize_text("café", 'NFC')
print(f"\nNormalized: {cafe_normalized!r}")

# Unicode categories
def get_category_counts(text: str) -> dict:
    from collections import Counter
    return Counter(unicodedata.category(c) for c in text)

sample = "Hello สวัสดี 123 !@#"
print(f"\nCategories in {sample!r}:")
for cat, count in sorted(get_category_counts(sample).items()):
    meaning = {'Lu': 'Upper', 'Ll': 'Lower', 'Lo': 'Other Letter',
               'Nd': 'Digit', 'Po': 'Punctuation', 'Zs': 'Space'}.get(cat, cat)
    print(f"  {cat} ({meaning}): {count}")
```

---

## 24.5 Emoji Detection

```python
import re

EMOJI = re.compile(
    '['
    '\U0001F300-\U0001F9FF'
    '\U00002702-\U000027B0'
    '\U000024C2-\U0001F251'
    ']+'
)

text = "Hello 👋 World 🌍 Python 🐍"
emojis = EMOJI.findall(text)
print(f"Emojis: {emojis}")

clean = EMOJI.sub('', text).strip()
print(f"Without emoji: {clean}")

tests = ["Hello", "Hello 🌍", "Python 🐍"]
for t in tests:
    print(f"  has_emoji({t!r}): {bool(EMOJI.search(t))}")
```

---

## 24.6 Multilingual Patterns

```python
import re

# Postal codes by country
POSTAL = {
    'TH': re.compile(r'\b[1-9]\d{4}\b'),
    'US': re.compile(r'\b\d{5}(?:-\d{4})?\b'),
    'UK': re.compile(r'\b[A-Z]{1,2}\d[A-Z\d]?\s?\d[A-Z]{2}\b', re.I),
    'JP': re.compile(r'\b\d{3}-\d{4}\b'),
}

addresses = [
    "Bangkok 10110 Thailand",
    "New York NY 10001 USA",
    "London EC1A 1BB UK",
    "Tokyo 150-0001 Japan",
]
print("Postal Codes:")
for addr in addresses:
    for country, p in POSTAL.items():
        m = p.search(addr)
        if m:
            print(f"  {addr}: {country} → {m.group()}")
            break

# Script detection
def detect_script(text: str) -> str:
    scripts = {
        'Thai':   r'[฀-๿]',
        'Arabic': r'[؀-ۿ]',
        'CJK':    r'[一-鿿]',
        'Latin':  r'[a-zA-Z]',
    }
    counts = {s: len(re.findall(p, text)) for s, p in scripts.items()}
    best = max(counts, key=counts.get)
    return best if counts[best] > 0 else 'Unknown'

samples = ["สวัสดี", "Hello", "こんにちは", "مرحبا"]
for s in samples:
    print(f"  {s!r}: {detect_script(s)}")
```

---

## 24.7 สรุป Part 24

```
Unicode Regex:
\w, \d, \s   — Unicode-aware by default (Python 3)
re.ASCII     — จำกัดเฉพาะ ASCII

Thai ranges:
[฀-๿]  — All Thai
[๐-๙]  — Thai digits ๐-๙

Best practices:
- unicodedata.normalize('NFC', text) ก่อน match
- ใช้ regex module สำหรับ \p{L}, \p{Thai}
- Emoji: \U0001F300-\U0001F9FF range
- Bytes pattern: rb'...' สำหรับ binary data
```

---

*[← Part 23: Regex Performance](part-23-regex-performance.md) | [→ Part 25: File Processing](part-25-file-processing.md)*
