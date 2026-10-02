# Part 64: Unicode & Internationalization with Regex

> **ระดับ:** สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-63

---

## 64.1 Unicode Fundamentals in Python Regex

```python
import re
import unicodedata
from typing import List, Dict

print("Unicode & Internationalization with Regex:")
print("=" * 60)

print("\n1. Unicode flags and properties:")

TEXT_EN = "Hello World 123"
TEXT_TH = "สวัสดี โลก ๑๒๓"
TEXT_JP = "こんにちは 世界 123"
TEXT_AR = "مرحبا بالعالم ١٢٣"
TEXT_MX = "H\xe9llo W\xf6rld 456"

WORD_RE   = re.compile(r'\w+')
LETTER_RE = re.compile(r'[^\W\d_]+')
DIGIT_RE  = re.compile(r'\d+')

print(f"\n   {'Language':<15} {'\\\\w+ tokens':<30} {'\\\\d+ tokens'}")
print(f"   {'-'*15} {'-'*30} {'-'*15}")
for lang, text in [('English', TEXT_EN), ('Thai', TEXT_TH),
                   ('Japanese', TEXT_JP), ('Arabic', TEXT_AR), ('Accented', TEXT_MX)]:
    words  = WORD_RE.findall(text)
    digits = DIGIT_RE.findall(text)
    print(f"   {lang:<15} {str(words):<30} {digits}")

print("\n\n2. Unicode categories via \\p{} (regex module):")

try:
    import regex
    LETTER_P = regex.compile(r'\p{L}+')
    NUMBER_P = regex.compile(r'\p{N}+')
    PUNCT_P  = regex.compile(r'\p{P}+')
    mixed = "Hello! สวัสดี 123 ١٢٣ \xa1Hola!"
    print(f"\n   Text: {mixed!r}")
    print(f"   Letters:     {LETTER_P.findall(mixed)}")
    print(f"   Numbers:     {NUMBER_P.findall(mixed)}")
    print(f"   Punctuation: {PUNCT_P.findall(mixed)}")
except ImportError:
    print("\n   (regex module needed for \\p{} Unicode property)")
    print("   Standard re module: \\w matches Unicode letters+digits+_")
    demo = "สวัสดี ๑๒๓ abc 123"
    print(f"\n   Demo with re: \\w+ on {demo!r}:")
    print(f"   {WORD_RE.findall(demo)}")
```

---

## 64.2 Thai Language Patterns

```python
import re
from typing import List, Dict

print("\nThai Language Patterns:")
print("=" * 60)

THAI_WORD    = re.compile(r'[฀-๿]+')
THAI_DIGIT   = re.compile(r'[๐-๙]+')
THAI_PHONE   = re.compile(r'\b0[6-9]\d{8}\b')
THAI_ID_CARD = re.compile(r'\b\d{13}\b')
THAI_POSTAL  = re.compile(r'\b[1-9]\d{4}\b')

THAI_DATE = re.compile(
    r'(?P<day>\d{1,2})\s+'
    r'(?P<month>มกราคม|กุมภาพันธ์|มีนาคม|เมษายน|พฤษภาคม|มิถุนายน|'
    r'กรกฎาคม|สิงหาคม|กันยายน|ตุลาคม|พฤศจิกายน|ธันวาคม)'
    r'\s+(?:พ\.ศส์\.|ค\.ศส์\.)\s*(?P<year>\d{4})'
)

THAI_BAHT = re.compile(
    r'(?:฿|บาท)\s*[\d,]+(?:\.\d{2})?|[\d,]+(?:\.\d{2})?\s*บาท'
)


def analyze_thai(text: str) -> Dict:
    return {
        'words':     THAI_WORD.findall(text),
        'digits_th': THAI_DIGIT.findall(text),
        'phones':    THAI_PHONE.findall(text),
        'postal':    THAI_POSTAL.findall(text),
        'amounts':   THAI_BAHT.findall(text),
    }


thai_texts = [
    "สวัสดีครับ กรุณาติดต่อ 0891234567 หรือ 0812345678",
    "ราคาสินค้า ฿1,250.50 หรือ 1250 บาท ส่งฟรี",
    "ที่อยู่: 123 ถนนสุขุมวิท กรุงเทพฯ 10110",
]

for text in thai_texts:
    result = analyze_thai(text)
    print(f"\n   Text: {text}")
    for k, v in result.items():
        if v:
            print(f"   {k:<12}: {v}")

print(f"\n   Thai date pattern test:")
date_texts = [
    "วันที่ 15 ตุลาคม พ.ศส์. 2567",
]
for dt in date_texts:
    m = THAI_DATE.search(dt)
    if m:
        print(f"   {dt!r}")
        print(f"   ➹ day={m.group('day')}, month={m.group('month')}, year={m.group('year')}")
    else:
        print(f"   {dt!r} → no match (pattern simplified for demo)")
```

---

## 64.3 Multilingual Text Processing

```python
import re
import unicodedata
from typing import Dict, List

print("\nMultilingual Text Processing:")
print("=" * 60)

def detect_language(text: str) -> str:
    samples = {
        'thai':     r'[฀-๿]',
        'arabic':   r'[؀-ۿ]',
        'chinese':  r'[一-鿿]',
        'japanese': r'[぀-ヿ]',
        'korean':   r'[가-힯]',
        'latin':    r'[a-zA-Z]',
        'cyrillic': r'[Ѐ-ӿ]',
    }
    scores = {lang: len(re.findall(pat, text)) for lang, pat in samples.items()}
    scores = {k: v for k, v in scores.items() if v > 0}
    return max(scores, key=lambda k: scores[k]) if scores else 'unknown'

def normalize_unicode(text: str) -> str:
    return unicodedata.normalize('NFC', text)

def remove_diacritics(text: str) -> str:
    decomposed = unicodedata.normalize('NFD', text)
    return ''.join(c for c in decomposed if unicodedata.category(c) != 'Mn')

def strip_zero_width(text: str) -> str:
    ZWC = re.compile(
        r'[​‌‍‎‏'
        r'﻿­⁠⁡⁢⁣⁤]'
    )
    return ZWC.sub('', text)


multilingual_samples = [
    "Hello World", "สวัสดีโลก",
    "مرحبا بالعالم",
    "你好世界", "こんにちは世界",
    "Привет мир", "H\xe9llo W\xf6rld",
]

print(f"\n   Language detection:")
print(f"   {'Text':<25} {'Detected'}")
print(f"   {'-'*25} {'-'*12}")
for text in multilingual_samples:
    lang = detect_language(text)
    print(f"   {text:<25} {lang}")

print(f"\n   Unicode normalization:")
texts_to_norm = ["caf\xe9", "na\xefve", "r\xe9sum\xe9", "\xc5ngstr\xf6m"]
for text in texts_to_norm:
    nfc    = normalize_unicode(text)
    nodiac = remove_diacritics(text)
    print(f"   {text!r:<15} → NFC: {nfc!r:<15} No-diacritics: {nodiac!r}")

print(f"\n   Zero-width character removal:")
obfuscated = "hel​lo‌ wor‍ld"
clean = strip_zero_width(obfuscated)
print(f"   Original (repr): {obfuscated!r}")
print(f"   Cleaned (repr):  {clean!r}")
print(f"   Cleaned display: {clean}")
```

---

## 64.4 Internationalized Input Validation

```python
import re
from typing import Dict, Optional

print("\nInternationalized Input Validation:")
print("=" * 60)

INTL_NAME = re.compile(
    r'^[ À-ɏ฀-๿一-鿿'
    r'぀-ヿ가-힯؀-ۿ'
    r'a-zA-Z\'\-]{2,100}$'
)

E164_PHONE = re.compile(r'^\+[1-9]\d{6,14}$')

THAI_NATIONAL_ID = re.compile(r'^\d{13}$')


def validate_thai_id(id_num: str) -> bool:
    if not THAI_NATIONAL_ID.match(id_num):
        return False
    digits = [int(d) for d in id_num]
    total  = sum(digits[i] * (13 - i) for i in range(12))
    check  = (11 - (total % 11)) % 10
    return check == digits[12]


validation_tests = {
    'name': [
        ('Alice Smith', True),
        ('สมชาย ใจดี', True),
        ('Mar\xeda Garc\xeda', True),
        ('山田太郎', True),
        ('AB', False),
        ('A' * 101, False),
    ],
    'phone_e164': [
        ('+66891234567', True),
        ('+14155552671', True),
        ('+441234567890', True),
        ('0891234567', False),
        ('+1234', False),
    ],
    'thai_id': [
        ('3100600785635', True),
        ('1234567890123', False),
        ('12345678901', False),
    ],
}

patterns = {'name': INTL_NAME, 'phone_e164': E164_PHONE}

for field, cases in validation_tests.items():
    print(f"\n   {field.upper()} validation:")
    for value, expected in cases:
        if field == 'thai_id':
            ok = validate_thai_id(value)
        else:
            ok = bool(patterns[field].match(value))
        status = '✓' if ok == expected else '✗ FAIL'
        val_str = value if len(value) < 25 else value[:22] + '...'
        print(f"   {status} {val_str:<30} expected={expected}, got={ok}")
```

---

## 64.5 สรุป Part 64

```
Unicode & Internationalization Regex:

1. Python 3 Unicode defaults:
   \w → matches Unicode letters+digits+_
   \d → matches Unicode digits (including Thai ๐-๙)
   re.UNICODE is the default flag

2. Thai character ranges:
   Block:       ฀-๿
   Consonants:  ก-ฮ
   Vowels:      ะ-ๅ
   Tone marks:  ่-๋
   Digits:      ๐-๙

3. Language detection:
   Check character ranges per language block
   Return the script with most matches

4. Unicode normalization:
   NFC: composed form (precomposed characters)
   NFD: decomposed (base + combining marks)
   Remove diacritics: filter category 'Mn'

5. Zero-width characters:
   ​-‏, ﻿, ⁠-⁤
   Used in text obfuscation → strip them

6. International phone: E.164 = ^\+[1-9]\d{6,14}$
```

---

*[← Part 63: Advanced Pattern Matching](part-63-advanced-patterns.md) | [→ Part 65: File & Path Processing](part-65-file-paths.md)*
