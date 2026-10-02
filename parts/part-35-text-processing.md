# Part 35: Text Processing — การประมวลผลข้อความ

> **ระดับ:** กลาง | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-34

---

## 35.1 Text Normalization

```python
import re
import unicodedata
from typing import List, Dict

class TextNormalizer:
    """Normalize text for processing"""
    
    # Whitespace
    MULTIPLE_SPACES   = re.compile(r' {2,}')
    MULTIPLE_NEWLINES = re.compile(r'\n{3,}')
    TRAILING_WHITESPACE = re.compile(r'[ \t]+$', re.MULTILINE)
    
    # Punctuation normalization
    SMART_QUOTES_OPEN  = re.compile(r'[‘’‚‛]')
    SMART_QUOTES_CLOSE = re.compile(r'[“”„‟]')
    DASHES   = re.compile(r'[–—―]')
    ELLIPSIS = re.compile(r'…|\.{3,}')
    
    # Thai-specific
    THAI_REPEATS = re.compile(r'([฀-๿])\1{2,}')
    THAI_DOUBLE_VOWEL = re.compile(r'([เ-ไ])\1+')
    
    URL   = re.compile(r'https?://\S+|www\.\S+')
    EMAIL = re.compile(r'\b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b')
    
    ARABIC_DIGITS = '0123456789'
    THAI_DIGITS   = '๐๑๒๓๔๕๖๗๘๙'
    
    @classmethod
    def normalize_whitespace(cls, text: str) -> str:
        text = cls.TRAILING_WHITESPACE.sub('', text)
        text = cls.MULTIPLE_SPACES.sub(' ', text)
        text = cls.MULTIPLE_NEWLINES.sub('\n\n', text)
        return text.strip()
    
    @classmethod
    def normalize_quotes(cls, text: str) -> str:
        text = cls.SMART_QUOTES_OPEN.sub("'", text)
        text = cls.SMART_QUOTES_CLOSE.sub('"', text)
        return text
    
    @classmethod
    def normalize_punctuation(cls, text: str) -> str:
        text = cls.normalize_quotes(text)
        text = cls.DASHES.sub('-', text)
        text = cls.ELLIPSIS.sub('...', text)
        return text
    
    @classmethod
    def thai_to_arabic(cls, text: str) -> str:
        """แปลงเลขไทยเป็นเลขอาระบิก: ๑๒๓ -> 123"""
        table = str.maketrans(cls.THAI_DIGITS, cls.ARABIC_DIGITS)
        return text.translate(table)
    
    @classmethod
    def arabic_to_thai(cls, text: str) -> str:
        """แปลงเลขอาระบิกเป็นเลขไทย: 123 -> ๑๒๓"""
        table = str.maketrans(cls.ARABIC_DIGITS, cls.THAI_DIGITS)
        return text.translate(table)
    
    @classmethod
    def remove_urls(cls, text: str) -> str:
        return cls.URL.sub('', text).strip()
    
    @classmethod
    def strip_html_tags(cls, text: str) -> str:
        return re.sub(r'<[^>]+>', '', text)
    
    @classmethod
    def normalize_thai(cls, text: str) -> str:
        text = cls.THAI_REPEATS.sub(r'\1\1', text)
        text = cls.THAI_DOUBLE_VOWEL.sub(r'\1', text)
        return unicodedata.normalize('NFC', text)
    
    @classmethod
    def normalize_all(cls, text: str) -> str:
        text = cls.strip_html_tags(text)
        text = cls.normalize_punctuation(text)
        text = cls.normalize_thai(text)
        text = cls.normalize_whitespace(text)
        return text


# ทดสอบ
normalizer = TextNormalizer

samples = [
    "Hello   World\n\n\n\nNext paragraph",
    "He said “Hello” and she replied ‘Hi’",
    "สวัสดีครับบบบบ มีปัญหาอะไรไหมครับ",
    "<p>This is <strong>HTML</strong> content</p>",
    "Visit us at https://example.com or email support@test.org",
]

print("Text Normalization:")
print("=" * 60)
for text in samples:
    normalized = normalizer.normalize_all(text)
    print(f"  Input:  {repr(text[:55])}")
    print(f"  Output: {repr(normalized[:55])}\n")

print("Thai number conversion:")
print(f"  Thai->Arabic: ๑,๒๓๔.๕๖ -> {normalizer.thai_to_arabic('๑,๒๓๔.๕๖')}")
print(f"  Arabic->Thai: 1,234.56 -> {normalizer.arabic_to_thai('1,234.56')}")
```

---

## 35.2 Text Extraction Patterns

```python
import re
from typing import List, Dict, Optional

class TextExtractor:
    """Extract structured data from unstructured text"""
    
    # Thai address components
    HOUSE_NUMBER = re.compile(r'(?:เลขที่\s*)?(\d+(?:/\d+)?)')
    MOO      = re.compile(r'หมู่(?:ที่)?\s*(\d+)')
    SOI      = re.compile(r'(?:ซอย|Soi)\s*([฀-๿a-zA-Z0-9\s]+?)(?=\s+(?:ถนน|แขวง|ตำบล|เขต|อำเภอ)|\s*$)')
    ROAD     = re.compile(r'(?:ถนน|ถง.|Road|Rd\.)\s*([฀-๿a-zA-Z\s]+?)(?=\s+(?:แขวง|ตำบล|เขต|อำเภอ)|\s*$)')
    TAMBON   = re.compile(r'(?:ตำบล|แขวง|ต\.)\s*([฀-๿]+)')
    AMPHOE   = re.compile(r'(?:อำเภอ|เขต|อ\.)\s*([฀-๿]+)')
    PROVINCE = re.compile(r'(?:จังหวัด|จ\.)\s*([฀-๿]+)')
    POSTAL   = re.compile(r'\b([1-9]\d{4})\b')
    
    # Financial amounts
    BAHT_AMOUNT = re.compile(
        r'(?:฿|บาท|THB)\s*([\d,]+(?:\.\d{2})?)'
        r'|([\d,]+(?:\.\d{2})?)\s*(?:บาท|THB)',
        re.IGNORECASE
    )
    
    TAX_ID = re.compile(r'\b(\d-\d{4}-\d{5}-\d{2}-\d)\b')
    
    @classmethod
    def _first(cls, pattern: re.Pattern, text: str) -> Optional[str]:
        m = pattern.search(text)
        return m.group(1).strip() if m else None
    
    @classmethod
    def extract_address(cls, text: str) -> Dict:
        return {
            'house_no':   cls._first(cls.HOUSE_NUMBER, text),
            'moo':        cls._first(cls.MOO, text),
            'soi':        cls._first(cls.SOI, text),
            'road':       cls._first(cls.ROAD, text),
            'tambon':     cls._first(cls.TAMBON, text),
            'amphoe':     cls._first(cls.AMPHOE, text),
            'province':   cls._first(cls.PROVINCE, text),
            'postal_code':cls._first(cls.POSTAL, text),
        }
    
    @classmethod
    def extract_amounts(cls, text: str) -> List[str]:
        amounts = []
        for m in cls.BAHT_AMOUNT.finditer(text):
            amount = m.group(1) or m.group(2)
            if amount:
                amounts.append(amount.replace(',', ''))
        return amounts


class KeyValueExtractor:
    """Extract key-value pairs from free text"""
    
    JSON_KV = re.compile(
        r'"(?P<key>[^"]+)"\s*:\s*(?:"(?P<str_val>[^"]*)"'
        r'|(?P<num_val>-?\d+(?:\.\d+)?)'
        r'|(?P<bool_val>true|false|null))'
    )
    EQ_KV = re.compile(r'(?P<key>[A-Z_][A-Z0-9_]*)=(?P<value>[^\s,;]+)')
    
    @classmethod
    def extract_json_fields(cls, text: str) -> Dict:
        result = {}
        for m in cls.JSON_KV.finditer(text):
            key = m.group('key')
            value = m.group('str_val') or m.group('num_val') or m.group('bool_val')
            result[key] = value
        return result
    
    @classmethod
    def extract_env_vars(cls, text: str) -> Dict:
        return {m.group('key'): m.group('value') for m in cls.EQ_KV.finditer(text)}


# ทดสอบ
extractor = TextExtractor

address_text = """
ที่อยู่: เลขที่ 99/1 หมู่ที่ 5 ซอยลาดพร้าว 80 ถนนลาดพร้าว
แขวงวังทองหลาง เขตวังทองหลาง กรุงเทพมหานคร 10310
"""

financial_text = """
ราคาสินค้า ฿1,250.00 บวกภาษี 125.00 บาท รวม 1,375 บาท
ค่าจัดส่ง THB 50.00 รวมทั้งสิ้น 1,425.00 บาท
"""

print("Thai Address Extraction:")
print("=" * 60)
addr = extractor.extract_address(address_text)
for k, v in addr.items():
    if v:
        print(f"  {k:12}: {v}")

print("\nFinancial Amounts:")
amounts = extractor.extract_amounts(financial_text)
print(f"  {amounts}")

kv = KeyValueExtractor
json_text = '{"user_id": 42, "name": "Alice", "active": true, "score": 98.5}'
print(f"\nJSON fields: {kv.extract_json_fields(json_text)}")
```

---

## 35.3 สรุป Part 35

```
Text Normalization:
Multiple spaces:    ' {2,}' -> ' '
Smart quotes:       [‘’] -> '  |  [“”] -> "
Em-dash:            [–—] -> '-'
Thai repeats:       ([฀-๿])\1{2,} -> \1\1
Thai number:        str.maketrans('๐๑๒๓๔๕๖๗๘๙', '0123456789')

Thai Address:
House:    เลขที่\s*(\d+(?:/\d+)?)
Tambon:   (?:ตำบล|แขวง)\s*([฀-๿]+)
Amphoe:   (?:อำเภอ|เขต)\s*([฀-๿]+)
Province: (?:จังหวัด|จ\.)\s*([฀-๿]+)
Postal:   \b([1-9]\d{4})\b

Financial:
Baht: (?:฿|บาท|THB)\s*([\d,]+(?:\.\d{2})?)

Tips:
- ใช้ unicodedata.normalize('NFC', text) สำหรับ Thai text
- แปลงเลขไทยก่อน extract ตัวเลข
- ลำดับ pattern สำคัญ: ทำ code blocks ก่อน inline
```

---

*[← Part 34: Log Analysis](part-34-log-analysis.md) | [→ Part 36: Data Validation](part-36-data-validation.md)*
