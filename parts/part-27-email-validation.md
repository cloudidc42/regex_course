# Part 27: Email & Form Validation — การตรวจสอบข้อมูล

> **ระดับ:** กลาง | **เวลาเรียน:** ~65 นาที | **ข้อกำหนด:** Part 01-26

---

## 27.1 Email Validation ระดับต่างๆ

```python
import re

# Level 1: Minimal (ง่ายที่สุด)
EMAIL_MINIMAL = re.compile(r'.+@.+\..+')

# Level 2: Basic (พอใช้ได้)
EMAIL_BASIC = re.compile(r'[^@\s]+@[^@\s]+\.[^@\s]+')

# Level 3: Standard (เหมาะสมส่วนใหญ่)
EMAIL_STANDARD = re.compile(
    r'\b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b'
)

# Level 4: RFC 5322 compliant (ซับซ้อนมาก)
EMAIL_RFC5322 = re.compile(
    r"(?:[a-zA-Z0-9!#$%&'*+/=?^_`{|}~-]+"
    r"(?:\.[a-zA-Z0-9!#$%&'*+/=?^_`{|}~-]+)*"
    r'|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]'
    r'|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")'
    r'@(?:(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)+'
    r'[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?'
    r'|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}'
    r'(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?'
    r'|[a-zA-Z0-9-]*[a-zA-Z0-9]:'
    r'(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]'
    r'|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\])'
)

test_emails = [
    "user@example.com",          # valid
    "user.name+tag@domain.org",  # valid
    "user@subdomain.example.co.th",  # valid
    "admin@localhost",           # valid (RFC 5322) but unusual
    "user@",                     # invalid
    "@domain.com",               # invalid
    "no-at-sign",                # invalid
    "double@@at.com",            # invalid
    "user name@domain.com",      # invalid (space)
    "user@domain",               # invalid (no TLD)
    "",                          # invalid
]

print("Email Validation:")
print("=" * 60)
print(f"{'Email':35} {'Min':5} {'Basic':7} {'Std':5}")
print("-" * 60)
for email in test_emails:
    min_ok  = bool(EMAIL_MINIMAL.match(email))
    basic_ok = bool(EMAIL_BASIC.match(email))
    std_ok  = bool(EMAIL_STANDARD.match(email))
    mark = lambda b: '✓' if b else '✗'
    print(f"  {email:35} {mark(min_ok):5} {mark(basic_ok):7} {mark(std_ok):5}")
```

---

## 27.2 Form Field Validators

```python
import re
from typing import Tuple, Optional

class FormValidator:
    """Validators สำหรับ form fields ต่างๆ"""
    
    EMAIL    = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
    PHONE_TH = re.compile(r'^(?:\+?66|0)[6-9]\d{8}$')
    PHONE_INTL = re.compile(r'^\+?\d{1,3}[\s\-]?\(?\d{1,4}\)?[\s\-]?\d{1,4}[\s\-]?\d{1,9}$')
    
    THAI_ID   = re.compile(r'^\d-\d{4}-\d{5}-\d{2}-\d$')
    PASSPORT  = re.compile(r'^[A-Z]{1,2}\d{6,9}$', re.IGNORECASE)
    
    DATE_ISO  = re.compile(r'^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$')
    DATE_TH   = re.compile(r'^(0?[1-9]|[12]\d|3[01])/(0?[1-9]|1[0-2])/(\d{4})$')
    TIME_24H  = re.compile(r'^([01]?\d|2[0-3]):([0-5]\d)(?::([0-5]\d))?$')
    
    URL       = re.compile(r'^https?://[^\s/$.?#].[^\s]*$', re.IGNORECASE)
    USERNAME  = re.compile(r'^[a-zA-Z][a-zA-Z0-9_-]{2,29}$')
    
    STRONG_PW = re.compile(
        r'^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[!@#$%^&*])'
        r'(?!.*\s).{8,32}$'
    )
    
    CREDIT_CARD = re.compile(
        r'^(?:4\d{15}'          # Visa
        r'|5[1-5]\d{14}'        # Mastercard
        r'|3[47]\d{13}'         # Amex
        r'|6(?:011|5\d{2})\d{12}' # Discover
        r')$'
    )
    
    @classmethod
    def validate_credit_card(cls, number: str) -> Tuple[bool, str]:
        """Luhn algorithm validation"""
        digits = re.sub(r'\D', '', number)
        if not digits or not cls.CREDIT_CARD.match(digits):
            return False, "Invalid card number format"
        
        total = 0
        reverse = digits[::-1]
        for i, d in enumerate(reverse):
            n = int(d)
            if i % 2 == 1:
                n *= 2
                if n > 9:
                    n -= 9
            total += n
        
        if total % 10 != 0:
            return False, "Card number fails Luhn check"
        
        card_types = [
            (r'^4',            'Visa'),
            (r'^5[1-5]',       'Mastercard'),
            (r'^3[47]',        'Amex'),
            (r'^6(?:011|5)',   'Discover'),
        ]
        card_type = 'Unknown'
        for pattern, ctype in card_types:
            if re.match(pattern, digits):
                card_type = ctype
                break
        
        return True, f"Valid {card_type}"
    
    @classmethod
    def validate_thai_id(cls, id_str: str) -> Tuple[bool, str]:
        """Validate Thai national ID with checksum"""
        clean = re.sub(r'[-\s]', '', id_str)
        if not re.match(r'^\d{13}$', clean):
            return False, "Must be 13 digits"
        
        total = sum(int(d) * (13-i) for i, d in enumerate(clean[:12]))
        check = (11 - (total % 11)) % 10
        
        if check != int(clean[12]):
            return False, f"Invalid checksum (expected {check})"
        
        return True, "Valid Thai ID"


# ทดสอบ
print("Form Validation:")
print("=" * 60)

test_cases = [
    ("Email",    FormValidator.EMAIL,    ["user@example.com", "invalid", "a@b.c"]),
    ("Phone TH", FormValidator.PHONE_TH, ["0891234567", "+66891234567", "123", "02345678"]),
    ("Date ISO", FormValidator.DATE_ISO, ["2026-10-02", "2026-13-01", "2026-00-10"]),
    ("Username", FormValidator.USERNAME, ["alice", "u", "valid_user-123", "has space"]),
    ("Password", FormValidator.STRONG_PW, ["P@ssw0rd", "weak", "NoSpecial1", "Good$Pass1"]),
]

for name, pattern, tests in test_cases:
    print(f"\n  {name}:")
    for t in tests:
        ok = bool(pattern.match(t))
        print(f"    {'✓' if ok else '✗'} {t!r}")

print("\n  Credit Cards:")
cards = ["4111111111111111", "5500000000000004", "378282246310005", "1234567890123456"]
for card in cards:
    ok, msg = FormValidator.validate_credit_card(card)
    print(f"    {'✓' if ok else '✗'} {card} — {msg}")

print("\n  Thai IDs:")
thai_ids = ["3100100123456", "1234567890123", "3-1001-00123-45-6"]
for tid in thai_ids:
    ok, msg = FormValidator.validate_thai_id(tid)
    print(f"    {'✓' if ok else '✗'} {tid} — {msg}")
```

---

## 27.3 Address Validation (Thailand)

```python
import re
from typing import Dict, Optional

class ThaiAddressParser:
    """Parse ที่อยู่ภาษาไทย"""
    
    POSTAL_CODE = re.compile(r'\b([1-9]\d{4})\b')
    HOUSE_NUM   = re.compile(r'(?:บ้านเลขที่\s*)?(\d+(?:/\d+)?(?:\s*[ก-ฮ])?)')
    MOO         = re.compile(r'หมู่\s*(?:ที่\s*)?(\d+)')
    SOI         = re.compile(r'ซอย\s*([\w฀-๿]+(?:\s+\d+)?)')
    ROAD        = re.compile(r'ถนน\s*([\w฀-๿]+)')
    TAMBON      = re.compile(r'ตำบล\s*([฀-๿]+)')
    AMPHOE      = re.compile(r'อำเภอ\s*([฀-๿]+)')
    CHANGWAT    = re.compile(r'จังหวัด\s*([฀-๿]+)')
    
    @classmethod
    def parse(cls, address: str) -> Dict:
        result = {}
        
        for name, pattern in [
            ('postal_code', cls.POSTAL_CODE),
            ('house_number', cls.HOUSE_NUM),
            ('moo', cls.MOO),
            ('soi', cls.SOI),
            ('road', cls.ROAD),
            ('tambon', cls.TAMBON),
            ('amphoe', cls.AMPHOE),
            ('changwat', cls.CHANGWAT),
        ]:
            m = pattern.search(address)
            if m:
                result[name] = m.group(1).strip()
        
        return result


# ทดสอบ
addresses = [
    "บ้านเลขที่ 123/45 หมู่ที่ 3 ซอยสุขุมวิท 21 ถนนสุขุมวิท ตำบลคลองเตย อำเภอคลองเตย จังหวัดกรุงเทพมหานคร 10110",
    "456 ถนนเพชรบุรี แขวงถนนเพชรบุรี เขตราชเทวี กรุงเทพฯ 10400",
    "789/1 หมู่ 5 ซอยนิมมาน ถนนนิมมานเหมินท์ ตำบลสุเทพ อำเภอเมือง จังหวัดเชียงใหม่ 50200",
]

print("Thai Address Parser:")
print("=" * 60)
for addr in addresses:
    print(f"\n  Address: {addr[:60]}...")
    parsed = ThaiAddressParser.parse(addr)
    for k, v in parsed.items():
        print(f"    {k:15}: {v}")
```

---

## 27.4 Input Sanitization

```python
import re

class InputSanitizer:
    """Sanitize user input"""
    
    SQL_INJECTION = re.compile(
        r"(?:'|\"|;|--|/\*|\*/|xp_|UNION|SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC)",
        re.IGNORECASE
    )
    
    XSS = re.compile(
        r'<\s*script|javascript\s*:|vbscript\s*:|on\w+\s*=|expression\s*\(',
        re.IGNORECASE
    )
    
    PATH_TRAVERSAL = re.compile(r'\.\.[\/\\]|[\/\\]\.\.') 
    
    CONTROL_CHARS = re.compile(r'[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]')
    
    @classmethod
    def has_sql_injection(cls, text: str) -> bool:
        return bool(cls.SQL_INJECTION.search(text))
    
    @classmethod
    def has_xss(cls, text: str) -> bool:
        return bool(cls.XSS.search(text))
    
    @classmethod
    def has_path_traversal(cls, path: str) -> bool:
        return bool(cls.PATH_TRAVERSAL.search(path))
    
    @classmethod
    def sanitize_username(cls, username: str) -> str:
        cleaned = re.sub(r'[^\w-]', '', username)
        cleaned = re.sub(r'^[-_]+|[-_]+$', '', cleaned)
        return cleaned[:50]
    
    @classmethod
    def sanitize_filename(cls, filename: str) -> str:
        filename = re.sub(r'.*[\/\\]', '', filename)
        filename = re.sub(r'[<>:"/\\|?*\x00-\x1f]', '_', filename)
        filename = re.sub(r'^\.+', '', filename)
        return filename[:255] or 'unnamed'
    
    @classmethod
    def sanitize_html(cls, text: str) -> str:
        return (text
            .replace('&', '&amp;')
            .replace('<', '&lt;')
            .replace('>', '&gt;')
            .replace('"', '&quot;')
            .replace("'", '&#39;'))


# ทดสอบ
print("Input Sanitization:")
print("=" * 60)

tests = {
    "SQL injection": [
        ("normal search query", False, 'sql'),
        ("'; DROP TABLE users; --", True, 'sql'),
        ("UNION SELECT * FROM passwords", True, 'sql'),
        ("name LIKE '%test%'", True, 'sql'),
    ],
    "XSS": [
        ("<p>Normal HTML</p>", False, 'xss'),
        ("<script>alert(1)</script>", True, 'xss'),
        ('<img onerror="alert(1)">', True, 'xss'),
        ("javascript:void(0)", True, 'xss'),
    ],
    "Path traversal": [
        ("/home/user/file.txt", False, 'path'),
        ("../../../etc/passwd", True, 'path'),
        ("..\\..\\windows\\system32", True, 'path'),
        ("/var/log/app.log", False, 'path'),
    ],
}

sanitizer = InputSanitizer
for category, cases in tests.items():
    print(f"\n  {category}:")
    for text, expected_bad, check_type in cases:
        if check_type == 'sql':
            is_bad = sanitizer.has_sql_injection(text)
        elif check_type == 'xss':
            is_bad = sanitizer.has_xss(text)
        else:
            is_bad = sanitizer.has_path_traversal(text)
        
        icon = '⚠️ ' if is_bad else '✓ '
        label = 'DETECTED' if is_bad else 'SAFE'
        print(f"    {icon} [{label}] {text[:50]!r}")

print("\n  Sanitize functions:")
print(f"  Username: {'admin<script>'!r} -> {sanitizer.sanitize_username('admin<script>')!r}")
print(f"  Filename: {'../../../etc/passwd'!r} -> {sanitizer.sanitize_filename('../../../etc/passwd')!r}")
print(f"  HTML:     {'<b>Bold & \"quoted\"</b>'!r} -> {sanitizer.sanitize_html('<b>Bold & \"quoted\"</b>')!r}")
```

---

## 27.5 สรุป Part 27

```
Validation Patterns:
Email:       ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
Phone TH:    ^(?:\+?66|0)[6-9]\d{8}$
Date ISO:    ^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$
Thai ID:     ^\d-\d{4}-\d{5}-\d{2}-\d$
Username:    ^[a-zA-Z][a-zA-Z0-9_-]{2,29}$
URL:         ^https?://[^\s/$.?#].[^\s]*$
Password:    ^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[!@#$%^&*]).{8,}$

Security:
SQL inj:     '|"|;|--|UNION|SELECT|DROP|EXEC (case insensitive)
XSS:         <script|javascript:|on\w+=|expression(
Path trav:   \.\.[\/\\]

Always:
- Server-side validation เสมอ
- Client-side regex เป็นแค่ UX improvement
- ใช้ prepared statements กับ SQL ไม่ใช่ regex
```

---

*[← Part 26: Web Scraping](part-26-web-scraping.md) | [→ Part 28: Database Patterns](part-28-database-patterns.md)*
