# Part 44: Unicode & Internationalization — Regex กับภาษาต่างๆ

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-43

---

## 44.1 Unicode Basics ใน Python

```python
import re
import unicodedata

print("Unicode in Python Regex:")
print("=" * 60)

# ===== Unicode flags =====
# re.UNICODE (default ใน Python 3)
# re.ASCII  → \\w, \\d, \\s ตาม ASCII เท่านั้น

text_mixed = "Hello สวัสดี 你好 مرحبا 123"

print("\n1. \\w with re.UNICODE vs re.ASCII:")
unicode_words = re.findall(r'\w+', text_mixed)
ascii_words   = re.findall(r'\w+', text_mixed, re.ASCII)
print(f"   re.UNICODE: {unicode_words}")
print(f"   re.ASCII:   {ascii_words}")

print("\n2. \\d (digits):")
digits_text = "Thai: ๐๑๒๓๔ Arabic: 0123 Devanagari: ०१२"
print(f"   re.UNICODE: {re.findall(r'\\d', digits_text)}")
print(f"   re.ASCII:   {re.findall(r'\\d', digits_text, re.ASCII)}")

# ===== Unicode character categories =====
print("\n3. Unicode Character Categories:")
CATEGORIES = {
    r'\w':  "Word char (Unicode includes Thai, CJK, etc.)",
    r'\d':  "Digit (Unicode includes ๐-๙, ०-९, etc.)",
    r'\s':  "Whitespace (includes \\u00a0, \\u3000, etc.)",
    r'\W':  "Non-word",
    r'\D':  "Non-digit",
}
sample = "A1 ก1 ๑   　"
for pattern, desc in CATEGORIES.items():
    found = re.findall(pattern, sample)
    print(f"   {pattern}  {desc}: {found}")

# ===== Unicode escapes in pattern =====
print("\n4. Unicode Escapes in Pattern:")
unicode_patterns = [
    (r'฀-๿', "Thai Unicode range (in [ ])"),
    (r'一-鿿', "CJK Unified Ideographs"),
    (r'؀-ۿ', "Arabic"),
    (r'ऀ-ॿ', "Devanagari"),
    (r'Ͱ-Ͽ', "Greek"),
]

# ใช้ใน character class
thai_pattern = re.compile(r'[฀-๿]+')
cjk_pattern  = re.compile(r'[一-鿿]+')

test_texts = [
    "ยินดีต้อนรับ Python",
    "你好世界 Hello",
    "مرحبا World",
    "Ελληνικά Greek",
]

print("   Thai extraction:")
for t in test_texts:
    thai = thai_pattern.findall(t)
    cjk  = cjk_pattern.findall(t)
    print(f"     {t!r:<40}")
    if thai: print(f"       Thai: {thai}")
    if cjk:  print(f"       CJK:  {cjk}")
```

---

## 44.2 Unicode Normalization

```python
import re
import unicodedata

print("\nUnicode Normalization:")
print("=" * 60)

# ===== Normalization forms =====
# NFC:  Canonical Decomposition + Canonical Composition  (most common)
# NFD:  Canonical Decomposition
# NFKC: Compatibility Decomposition + Canonical Composition
# NFKD: Compatibility Decomposition

text_examples = [
    "café",       # e + combining accent
    "ﬁle",        # ﬁ = fi ligature
    "２０２４",    # fullwidth digits
    "１２３ＡＢＣ", # fullwidth alphanumeric
]

print("\n1. Normalization comparison:")
for text in text_examples:
    nfc  = unicodedata.normalize('NFC',  text)
    nfkc = unicodedata.normalize('NFKC', text)
    print(f"   Original: {text!r:<20} (len={len(text)})")
    print(f"   NFC:      {nfc!r:<20} (len={len(nfc)})")
    print(f"   NFKC:     {nfkc!r:<20} (len={len(nfkc)})")
    print()

# ===== Thai text normalization =====
print("2. Thai Text Normalization:")

THAI_VOWEL_ORDER = re.compile(r'([เ-ไ])([ก-ฮ])')

def normalize_thai_vowels(text: str) -> str:
    """แก้ vowel ที่อยู่ผิดตำแหน่ง (leading vowels stored differently)"""
    return text  # Unicode standard จัดการแล้ว

def remove_thai_tone_marks(text: str) -> str:
    """ลบวรรณยุกต์ไทย"""
    # Thai tone marks: ่ ้ ๊ ๋ ็
    TONE_MARKS = re.compile(r'[่-๋็]')
    return TONE_MARKS.sub('', text)

def remove_thai_diacritics(text: str) -> str:
    """ลบ diacritics ไทย (วรรณยุกต์ + สระบน)"""
    THAI_DIACRITICS = re.compile(r'[ะ-๎]')
    return THAI_DIACRITICS.sub('', text)

thai_samples = [
    "สวัสดีครับ",
    "ยิ้มแย้มแจ่มใส",
    "โปรแกรมภาษาไทย",
]

for t in thai_samples:
    no_tone = remove_thai_tone_marks(t)
    print(f"   Original: {t}")
    print(f"   No tones: {no_tone}")
    print()

# ===== Fullwidth to ASCII conversion =====
print("3. Fullwidth → ASCII Normalization:")

FULLWIDTH_DIGIT = re.compile(r'[０-９]')
FULLWIDTH_ALPHA = re.compile(r'[Ａ-Ｚａ-ｚ]')

def normalize_fullwidth(text: str) -> str:
    """Convert fullwidth chars to ASCII"""
    result = unicodedata.normalize('NFKC', text)
    return result

fullwidth_tests = [
    "２０２４年１２月３１日",
    "ＰＹＴＨＯＮ３．１２",
    "ＩＤ：１２３４５",
    "Ｈｅｌｌｏ　Ｗｏｒｌｄ",  # fullwidth space
]

for t in fullwidth_tests:
    normalized = normalize_fullwidth(t)
    print(f"   {t!r:<30}")
    print(f"   → {normalized!r}")
    print()
```

---

## 44.3 Regex สำหรับภาษาต่างๆ

```python
import re
from typing import List, Dict

print("\nMultilingual Regex Patterns:")
print("=" * 60)

# ===== Thai Language Patterns =====
class ThaiPatterns:
    # ตัวอักษรไทย (ไม่รวม tone marks)
    CONSONANTS = re.compile(r'[ก-ฮ]+')
    
    # คำไทย (พยัญชนะ + สระ + วรรณยุกต์)
    WORD = re.compile(r'[฀-๿]+')
    
    # เลขไทย
    DIGITS = re.compile(r'[๐-๙]+')
    
    # วันที่แบบไทย: ๑ มกราคม ๒๕๖๗ / 1 มกราคม 2567
    DATE = re.compile(
        r'(\d+|[๐-๙]+)\s+'
        r'(มกราคม|กุมภาพันธ์|มีนาคม|เมษายน|พฤษภาคม|มิถุนายน|'
        r'กรกฎาคม|สิงหาคม|กันยายน|ตุลาคม|พฤศจิกายน|ธันวาคม)\s+'
        r'(\d{4}|[๐-๙]{4})'
    )
    
    # หมายเลขโทรศัพท์ไทย
    PHONE = re.compile(r'(?:0|\+66)\s?[6-9]\d{8}')
    
    # รหัสไปรษณีย์ไทย
    POSTAL = re.compile(r'\b[1-9]\d{4}\b')
    
    @classmethod
    def extract_all(cls, text: str) -> Dict[str, List]:
        return {
            'words':   cls.WORD.findall(text),
            'digits':  cls.DIGITS.findall(text),
            'phones':  cls.PHONE.findall(text),
            'dates':   cls.DATE.findall(text),
        }


# ===== CJK (Chinese/Japanese/Korean) Patterns =====
class CJKPatterns:
    # CJK Unified Ideographs (Chinese characters)
    CHINESE = re.compile(r'[一-鿿㐀-䶿 0-⩭F]+')
    
    # Hiragana
    HIRAGANA = re.compile(r'[぀-ゟ]+')
    
    # Katakana
    KATAKANA = re.compile(r'[゠-ヿ]+')
    
    # Korean Hangul
    HANGUL = re.compile(r'[가-힯ᄀ-ᇿ]+')
    
    # Japanese (mix of CJK + Hiragana + Katakana)
    JAPANESE = re.compile(r'[぀-ヿ一-鿿]+')
    
    @classmethod
    def detect_script(cls, text: str) -> List[str]:
        """Detect which writing systems are in the text"""
        scripts = []
        if cls.CHINESE.search(text):   scripts.append('Chinese/CJK')
        if cls.HIRAGANA.search(text):  scripts.append('Hiragana')
        if cls.KATAKANA.search(text):  scripts.append('Katakana')
        if cls.HANGUL.search(text):    scripts.append('Korean')
        return scripts


# ===== Arabic/RTL Patterns =====
class ArabicPatterns:
    ARABIC_CHARS = re.compile(r'[؀-ۿ]+')
    ARABIC_NUM   = re.compile(r'[٠-٩]+')  # Eastern Arabic numerals
    
    @classmethod
    def extract_arabic(cls, text: str) -> List[str]:
        return cls.ARABIC_CHARS.findall(text)


# ===== Tests =====
print("\nThai Pattern Tests:")
thai_text = """
ติดต่อ: นางสาวสมหญิง ใจดี
โทร: 089-123-4567 หรือ +66812345678
วันเกิด: 15 มกราคม 2540
ที่อยู่: 123/45 ถนนสุขุมวิท แขวงคลองเตย เขตคลองเตย กรุงเทพมหานคร 10110
"""

thai = ThaiPatterns
result = thai.extract_all(thai_text)
for key, values in result.items():
    if values:
        print(f"  {key}: {values}")

print("\nCJK Pattern Tests:")
test_texts_cjk = [
    ("你好世界",      "Chinese"),
    ("こんにちは",     "Hiragana"),
    ("コンピュータ",  "Katakana"),
    ("안녕하세요",    "Korean"),
    ("日本語のテスト", "Japanese mixed"),
]

cjk = CJKPatterns
for text, lang in test_texts_cjk:
    detected = cjk.detect_script(text)
    print(f"  {text!r:<20} → {detected}")

print("\nArabic Pattern Tests:")
arabic_texts = [
    "مرحبا بالعالم",
    "البرمجة ٣١٢",
]
arabic = ArabicPatterns
for t in arabic_texts:
    found = arabic.extract_arabic(t)
    print(f"  {t!r:<25} → {found}")
```

---

## 44.4 Unicode Property Escapes (regex module)

```python
try:
    import regex
    print("\nUnicode Properties (regex module):")
    print("=" * 60)
    
    # Unicode property escapes: \p{Property}
    # Available: \p{L} = Letter, \p{N} = Number, \p{P} = Punctuation
    # \p{Lu} = Uppercase Letter, \p{Ll} = Lowercase Letter
    # \p{Script=Thai}, \p{Script=Latin}
    
    texts = [
        "Hello สวัสดี 你好 123 !@#",
        "Python3.11 ไพธอน Python-Dev",
    ]
    
    for text in texts:
        print(f"\n  Text: {text!r}")
        
        # All letters (any script)
        letters = regex.findall(r'\p{L}+', text)
        print(f"  \\p{{L}} (letters): {letters}")
        
        # All numbers (any script)
        numbers = regex.findall(r'\p{N}+', text)
        print(f"  \\p{{N}} (numbers): {numbers}")
        
        # Thai script specifically
        thai = regex.findall(r'\p{Script=Thai}+', text)
        print(f"  \\p{{Script=Thai}}: {thai}")
        
        # Latin script
        latin = regex.findall(r'\p{Script=Latin}+', text)
        print(f"  \\p{{Script=Latin}}: {latin}")
    
    # Unicode category breakdown
    print("\n  Unicode General Categories:")
    cats = {
        r'\p{Lu}': 'Uppercase Letter',
        r'\p{Ll}': 'Lowercase Letter',
        r'\p{Nd}': 'Decimal Digit',
        r'\p{Ps}': 'Open Punctuation [({',
        r'\p{Pe}': 'Close Punctuation )}]',
        r'\p{Sm}': 'Math Symbol',
        r'\p{So}': 'Other Symbol',
        r'\p{Zs}': 'Space Separator',
    }
    
    sample = "Hello, World! (Test) 2024 +-=≠ ∑ © → ☆"
    for pattern, name in cats.items():
        found = regex.findall(pattern, sample)
        if found:
            print(f"    {pattern:<12} {name:<25}: {found[:5]}")

except ImportError:
    print("\n  regex module ไม่ได้ติดตั้ง: pip install regex")
    print("  ใช้ Python re กับ Unicode range แทน: [\\u0E00-\\u0E7F]")
```

---

## 44.5 Practical: International Data Validation

```python
import re
from typing import Tuple, Dict

print("\nInternational Data Validation:")
print("=" * 60)

class InternationalValidator:
    
    # Email (RFC 5321 simplified, Unicode allowed)
    EMAIL = re.compile(
        r'^[a-zA-Z0-9._%+\-฀-๿]+@'
        r'[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$'
    )
    
    # International phone (E.164 format)
    PHONE_E164 = re.compile(r'^\+[1-9]\d{6,14}$')
    
    # Country-specific phones
    PHONES = {
        'TH': re.compile(r'^(?:0|\+66)[6-9]\d{8}$'),
        'US': re.compile(r'^(?:1|\+1)?[2-9]\d{2}[2-9]\d{6}$'),
        'JP': re.compile(r'^(?:0|\+81)[0-9]{9,10}$'),
        'UK': re.compile(r'^(?:0|\+44)[1-9]\d{9}$'),
        'SG': re.compile(r'^(?:65|\+65)[6-9]\d{7}$'),
        'CN': re.compile(r'^(?:0|\+86)1[3-9]\d{9}$'),
    }
    
    # Postal codes
    POSTAL = {
        'TH': re.compile(r'^[1-9]\d{4}$'),
        'US': re.compile(r'^\d{5}(?:-\d{4})?$'),
        'JP': re.compile(r'^\d{3}-?\d{4}$'),
        'UK': re.compile(r'^[A-Z]{1,2}\d[A-Z\d]? \d[A-Z]{2}$'),
        'SG': re.compile(r'^\d{6}$'),
    }
    
    # Currency amounts (with optional symbols)
    CURRENCY = re.compile(
        r'^(?:'
        r'(?P<symbol>[฿$€£¥₩])?\s*(?P<amount>[\d,]+(?:\.\d{1,2})?)'
        r'|(?P<amount2>[\d,]+(?:\.\d{1,2})?)\s*(?P<code>THB|USD|EUR|GBP|JPY|KRW|SGD)'
        r')$',
        re.IGNORECASE
    )
    
    @classmethod
    def validate_phone(cls, phone: str, country: str = 'TH') -> Tuple[bool, str]:
        clean = re.sub(r'[\s\-\(\)]', '', phone)
        pattern = cls.PHONES.get(country.upper())
        if not pattern:
            # Fallback: E.164
            if cls.PHONE_E164.match(clean):
                return True, "Valid E.164 format"
            return False, f"Unknown country code: {country}"
        
        if pattern.match(clean):
            return True, f"Valid {country} phone"
        return False, f"Invalid {country} phone format"
    
    @classmethod
    def validate_postal(cls, code: str, country: str) -> Tuple[bool, str]:
        pattern = cls.POSTAL.get(country.upper())
        if not pattern:
            return False, f"No pattern for country: {country}"
        clean = code.strip().upper()
        if pattern.match(clean):
            return True, f"Valid {country} postal code"
        return False, f"Invalid {country} postal code"
    
    @classmethod
    def parse_currency(cls, amount_str: str) -> Dict:
        m = cls.CURRENCY.match(amount_str.strip())
        if not m:
            return {'valid': False, 'raw': amount_str}
        
        d = m.groupdict()
        symbol = d.get('symbol')
        code   = d.get('code')
        amount = d.get('amount') or d.get('amount2')
        
        SYMBOL_MAP = {'฿': 'THB', '$': 'USD', '€': 'EUR', '£': 'GBP', '¥': 'JPY', '₩': 'KRW'}
        currency = code or (SYMBOL_MAP.get(symbol, '?') if symbol else 'UNKNOWN')
        
        try:
            value = float(amount.replace(',', ''))
        except (ValueError, AttributeError):
            value = None
        
        return {
            'valid': True,
            'value': value,
            'currency': currency,
            'formatted': f"{value:,.2f} {currency}" if value else None
        }


validator = InternationalValidator

print("\nPhone Validation:")
phone_tests = [
    ('0812345678', 'TH'),
    ('+66812345678', 'TH'),
    ('2125551234', 'US'),
    ('+12125551234', 'US'),
    ('090-1234-5678', 'JP'),
    ('invalid', 'TH'),
]

for phone, country in phone_tests:
    ok, msg = validator.validate_phone(phone, country)
    status = "✓" if ok else "✗"
    print(f"  {status} {phone:<20} ({country}): {msg}")

print("\nPostal Code Validation:")
postal_tests = [
    ('10110', 'TH'),
    ('10001', 'US'),
    ('10001-2345', 'US'),
    ('150-0001', 'JP'),
    ('SW1A 1AA', 'UK'),
    ('528000', 'SG'),
    ('99999', 'TH'),
]

for code, country in postal_tests:
    ok, msg = validator.validate_postal(code, country)
    status = "✓" if ok else "✗"
    print(f"  {status} {code:<15} ({country}): {msg}")

print("\nCurrency Parsing:")
currency_tests = [
    '฿1,250.00',
    '$9.99',
    '€1,200',
    '1250 THB',
    '99.99USD',
    '¥5000',
    'invalid',
]

for c in currency_tests:
    result = validator.parse_currency(c)
    if result['valid']:
        print(f"  ✓ {c!r:<15} → {result['formatted']}")
    else:
        print(f"  ✗ {c!r:<15} → invalid")
```

---

## 44.6 สรุป Part 44

```
Unicode ใน Python re:
- Python 3: re.UNICODE เป็น default
- \\w ครอบคลุมตัวอักษรไทย, CJK, Arabic ฯลฯ
- \\d ครอบคลุมเลขหลายภาษา (๐-๙, ०-९, ٠-٩)
- ใช้ re.ASCII เมื่อต้องการ ASCII เท่านั้น

Unicode Ranges สำคัญ:
\\u0E00-\\u0E7F  Thai
\\u4E00-\\u9FFF  CJK (Chinese/Japanese)
\\u3040-\\u309F  Hiragana
\\u30A0-\\u30FF  Katakana
\\uAC00-\\uD7AF  Korean Hangul
\\u0600-\\u06FF  Arabic
\\u0370-\\u03FF  Greek

Normalization:
NFC   → Web standard (แนะนำสำหรับ web)
NFKC  → เปลี่ยน fullwidth, ligatures เป็น ASCII
NFD   → Decomposed form (แยก base+combining)

regex module (pip install regex):
\\p{L}            = Any letter
\\p{N}            = Any number
\\p{Script=Thai}  = Thai script
\\p{Lu}           = Uppercase letter
\\p{Ll}           = Lowercase letter

Best Practices:
1. Normalize (NFKC) input ก่อน validate
2. ใช้ re.UNICODE (default) เสมอใน Python 3
3. Test ด้วย emoji, RTL text, combining chars
4. unicodedata.normalize() ก่อน compare
```

---

*[← Part 43: Engine Internals](part-43-engine-internals.md) | [→ Part 45: Multiline & DOTALL Flags](part-45-multiline-dotall.md)*
