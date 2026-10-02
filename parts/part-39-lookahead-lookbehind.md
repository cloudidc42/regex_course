# Part 39: Lookahead & Lookbehind — Zero-Width Assertions

> **ระดับ:** สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-38

---

## 39.1 Lookahead Assertions

```python
import re

print("Lookahead Assertions:")
print("=" * 60)

# ===== Positive Lookahead (?=...) =====
# Match position ที่มี pattern ต่อจากนี้ (ไม่ consume)

text = "cat9 dog bird2 fish"
result = re.findall(r'\b\w+(?=\d)', text)
print(f"\n  Words followed by digit: {result}")

files = "main.py utils.js config.yaml data.json test.py"
extensions = re.findall(r'\w+(?=\.(?:py|js|yaml|json))', files)
print(f"  Filenames before extension: {extensions}")

prices = "100 THB 200USD 300฿ 400EUR"
amounts_with_currency = re.findall(r'\d+(?=\s*(?:THB|USD|฿|EUR))', prices)
print(f"  Amounts before currency: {amounts_with_currency}")

# ===== Negative Lookahead (?!...) =====
result_neg = re.findall(r'\b\w+\b(?!\d)', text)
print(f"\n  Words NOT followed by digit: {result_neg}")

colors = "colorful colorless color colored"
not_colorful = re.findall(r'color(?!ful)\w*', colors)
print(f"  'color' not before 'ful': {not_colorful}")

# ===== Multiple lookaheads (password validation) =====
PW_PATTERNS = {
    'has_upper':   re.compile(r'(?=.*[A-Z])'),
    'has_lower':   re.compile(r'(?=.*[a-z])'),
    'has_digit':   re.compile(r'(?=.*\d)'),
    'has_special': re.compile(r'(?=.*[!@#$%^&*])'),
    'min_8_chars': re.compile(r'.{8,}'),
}

PW_STRONG = re.compile(
    r'^'
    r'(?=.*[A-Z])'
    r'(?=.*[a-z])'
    r'(?=.*\d)'
    r'(?=.*[!@#$%^&*])'
    r'.{8,}$'
)

passwords = ['weak', 'Stronger1', 'Str0ng!Pw', 'My$ecure1Pass', 'NoSpecial1234']
print("\n  Password Validation (lookahead):")
for pw in passwords:
    checks = {k: bool(v.search(pw)) for k, v in PW_PATTERNS.items()}
    strong = bool(PW_STRONG.match(pw))
    passed = sum(checks.values())
    print(f"    {pw:<20} {'STRONG' if strong else 'WEAK':6} ({passed}/5 checks)")
```

---

## 39.2 Lookbehind Assertions

```python
import re

print("\nLookbehind Assertions:")
print("=" * 60)

# ===== Positive Lookbehind (?<=...) =====
financial_text = "Price: ฿1250 Tax: ฿125 Total: ฿1375 Fee: $50"
baht_amounts = re.findall(r'(?<=฿)\d+', financial_text)
dollar_amounts = re.findall(r'(?<=\$)\d+', financial_text)
print(f"\n  Baht amounts:   {baht_amounts}")
print(f"  Dollar amounts: {dollar_amounts}")

text = "I love Python and I love JavaScript and I love regex"
after_love = re.findall(r'(?<=love )\w+', text)
print(f"  Things after 'love': {after_love}")

emails = "alice@gmail.com bob@yahoo.co.th charlie@company.org"
domains = re.findall(r'(?<=@)[\w.]+', emails)
print(f"  Email domains: {domains}")

# ===== Negative Lookbehind (?<!...) =====
verbs = "running jumping testing sitting coding"
non_ting = re.findall(r'\w+(?<!t)ing\b', verbs)
print(f"\n  '-ing' NOT after 't': {non_ting}")

numbers = "100 -200 300 -400 500"
positive_only = re.findall(r'(?<!-)\b\d+', numbers)
print(f"  Digits not after '-': {positive_only}")

# ===== Variable-length Lookbehind (regex module) =====
try:
    import regex
    
    log = "ERROR: something failed\nWARNING: check config\nINFO: server started\nERROR: disk full"
    error_msgs = regex.findall(r'(?<=ERROR: ).+', log)
    print(f"\n  Error messages (regex module): {error_msgs}")
    
    text = "USD100 EUR200 100JPY 300THB"
    amounts = regex.findall(r'(?<=[A-Z]{2,3})\d+', text)
    print(f"  Amounts after 2-3 char code: {amounts}")
    
except ImportError:
    print("\n  (regex module not available)")

# Workaround
def extract_after_pattern(text: str, before: str, target: str) -> list:
    full = re.findall(f'(?:{before})({target})', text)
    return full

data = "user ID:12345 order id:98765 product ID:11111"
ids = extract_after_pattern(data, r'[Ii][Dd]:\s*', r'\d+')
print(f"\n  IDs after 'ID:' or 'id:': {ids}")
```

---

## 39.3 Practical Lookaround Patterns

```python
import re
from typing import List

class LookaroundPatterns:
    """Collection ของ pattern ที่ใช้ lookaround"""
    
    ADD_COMMA = re.compile(r'(?<=\d)(?=(\d{3})+(?!\d))')
    CSS_VALUE = re.compile(r'(?<=:\s)[\w#%().,\s]+(?=;)')
    JSON_VALUE = re.compile(r'(?<="[^"]+"\s*:\s*)"([^"]*)"')
    VERSION = re.compile(r'(?<=(?:v|version\s))([\d]+\.[\d]+\.?[\d]*)', re.IGNORECASE)
    PASSWORD_CHECK = re.compile(
        r'^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^a-zA-Z\d]).{8,}$'
    )
    SPLIT_COMMA = re.compile(r',(?![^(]*\))')
    CAMEL_SPLIT = re.compile(r'(?<=[a-z])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])')
    
    @classmethod
    def add_thousands_comma(cls, number_str: str) -> str:
        return cls.ADD_COMMA.sub(',', number_str)
    
    @classmethod
    def extract_css_values(cls, css: str) -> List[str]:
        return cls.CSS_VALUE.findall(css)
    
    @classmethod
    def split_camel_case(cls, word: str) -> List[str]:
        return cls.CAMEL_SPLIT.split(word)
    
    @classmethod
    def split_outside_parens(cls, text: str) -> List[str]:
        return cls.SPLIT_COMMA.split(text)
    
    @classmethod
    def extract_versions(cls, text: str) -> List[str]:
        return cls.VERSION.findall(text)


patterns = LookaroundPatterns

print("\nPractical Lookaround Patterns:")
print("=" * 60)

numbers = ['1234', '1234567', '1000000000']
print("\n  Thousands comma:")
for n in numbers:
    print(f"    {n:<15} -> {patterns.add_thousands_comma(n)}")

css = "color: #ff0000; font-size: 16px; margin: 10px 20px; background: rgba(0,0,0,0.5);"
values = patterns.extract_css_values(css)
print(f"\n  CSS values: {values}")

camel_words = ['camelCase', 'HTTPSHandler', 'parseHTMLDocument', 'getUserByID']
print(f"\n  CamelCase split:")
for w in camel_words:
    parts = patterns.split_camel_case(w)
    print(f"    {w:<25} -> {parts}")

data = "func(a, b), method(x, y), simple_arg, another_func(1, 2, 3)"
parts = patterns.split_outside_parens(data)
print(f"\n  Split outside parens:")
for p in parts:
    print(f"    {p.strip()!r}")

version_texts = [
    "Python v3.11.0 and Flask v2.3.1",
    "version 1.0.0-beta",
    "webpack@5.88.2",
]
print(f"\n  Version extraction:")
for t in version_texts:
    vs = patterns.extract_versions(t)
    print(f"    {t!r:<45} -> {vs}")
```

---

## 39.4 สรุป Part 39

```
Lookaround Quick Reference:
(?=pattern)    positive lookahead   "ตามด้วย pattern"
(?!pattern)    negative lookahead   "ไม่ตามด้วย pattern"
(?<=pattern)   positive lookbehind  "อยู่หลัง pattern" [fixed-width]
(?<!pattern)   negative lookbehind  "ไม่อยู่หลัง pattern" [fixed-width]

Key: lookaround ไม่ consume characters (zero-width)

Practical uses:
- Password: ^(?=.*[A-Z])(?=.*\d).{8,}$
- Numbers:  (?<=฿)\d+  ←  amount after currency
- Split:    ,(?![^(]*\))  ←  comma outside parens
- Camel:    (?<=[a-z])(?=[A-Z])  ←  split point
- Comma:    (?<=\d)(?=(\d{3})+(?!\d))  ←  thousands

Limitations (Python re):
- lookbehind ต้องเป็น fixed-width: (?<=ab) OK, (?<=a+) ERROR
- แก้ด้วย regex module (pip install regex) สำหรับ variable-length

Common mistakes:
- ใส่ quantifier ใน lookbehind: (?<=\d+)  <- ERROR
- ลืม ^ anchor กับ multiple lookaheads
- confuse lookahead กับ match: (?=\d)\d+ <- redundant
```

---

*[← Part 38: Advanced Groups](part-38-advanced-groups.md) | [→ Part 40: Regex Performance](part-40-performance.md)*
