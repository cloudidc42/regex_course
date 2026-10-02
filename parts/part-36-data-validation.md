# Part 36: Data Validation — การตรวจสอบข้อมูล

> **ระดับ:** กลาง | **เวลาเรียน:** ~65 นาที | **ข้อกำหนด:** Part 01-35

---

## 36.1 Comprehensive Validation Library

```python
import re
from typing import Tuple, Optional, Dict, Any

class Validator:
    """Library สำหรับ validate ข้อมูลทุกรูปแบบ"""
    
    THAI_ID = re.compile(r'^\d-\d{4}-\d{5}-\d{2}-\d$')
    PASSPORT_THAI = re.compile(r'^[A-Z]{2}\d{7}$')
    
    PHONE_TH_MOBILE = re.compile(r'^(?:\+?66|0)(?:6[1-9]|8[1-9]|9[0-9])\d{7}$')
    PHONE_TH_HOME   = re.compile(r'^(?:\+?66|0)(?:2|3[2-9]|4[2-9]|5[2-9]|7[3-7])\d{7,8}$')
    PHONE_TH_ANY    = re.compile(r'^(?:\+?66|0)[6-9]\d{8}$|^(?:\+?66|0)[2-7]\d{7,8}$')
    
    EMAIL_STANDARD = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
    EMAIL_STRICT   = re.compile(
        r'^(?![.-])'
        r'[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~.-]{1,64}'
        r'(?<![.-])'
        r'@'
        r'(?![.-])'
        r'[a-zA-Z0-9.-]{1,253}'
        r'(?<![.])'
        r'\.[a-zA-Z]{2,}$'
    )
    
    CREDIT_CARD_VISA       = re.compile(r'^4\d{12}(?:\d{3})?$')
    CREDIT_CARD_MASTERCARD = re.compile(r'^5[1-5]\d{14}$')
    CREDIT_CARD_AMEX       = re.compile(r'^3[47]\d{13}$')
    CREDIT_CARD_JCB        = re.compile(r'^(?:3528|3529|35[2-8]\d)\d{12}$')
    
    BANK_ACCOUNT_TH = re.compile(r'^\d{3}-\d{1}-\d{5}-\d{1}$')
    SWIFT = re.compile(r'^[A-Z]{6}[A-Z0-9]{2}(?:[A-Z0-9]{3})?$')
    IBAN  = re.compile(r'^[A-Z]{2}\d{2}[A-Z0-9]{4,30}$')
    PROMPTPAY = re.compile(r'^(?:0\d{9}|\d-\d{4}-\d{5}-\d{2}-\d|[1-9]\d{9,12})$')
    
    UUID_V4  = re.compile(r'^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$', re.IGNORECASE)
    UUID_ANY = re.compile(r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$', re.IGNORECASE)
    SLUG     = re.compile(r'^[a-z0-9]+(?:-[a-z0-9]+)*$')
    USERNAME = re.compile(r'^[a-zA-Z][a-zA-Z0-9_.-]{2,29}$')
    
    IPV4 = re.compile(
        r'^(?:(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)\.){3}'
        r'(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)$'
    )
    
    DOMAIN = re.compile(
        r'^(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)*'
        r'[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?'
        r'\.[a-zA-Z]{2,}$'
    )
    
    URL = re.compile(
        r'^(?P<scheme>https?|ftp)://'
        r'(?P<host>[a-zA-Z0-9.-]+(?:\.[a-zA-Z]{2,})|localhost|\d{1,3}(?:\.\d{1,3}){3})'
        r'(?::(?P<port>\d{1,5}))?'
        r'(?P<path>/[^\s?#]*)?'
        r'(?:\?(?P<query>[^\s#]*))?'
        r'(?:#(?P<fragment>[^\s]*))?$'
    )
    
    PW_STRONG = re.compile(
        r'^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*()_+=\-\[\]{}|;:,.<>?]).{8,}$'
    )
    
    @classmethod
    def validate_thai_id(cls, id_str: str) -> Tuple[bool, str]:
        """ตรวจสอบเลขบัตรประชาชนไทยพร้อม checksum"""
        clean = re.sub(r'[-\s]', '', id_str)
        if len(clean) != 13 or not clean.isdigit():
            return False, "ต้องมี 13 หลัก"
        total = sum(int(clean[i]) * (13 - i) for i in range(12))
        check = (11 - (total % 11)) % 10
        if check != int(clean[12]):
            return False, f"checksum ไม่ถูกต้อง (ต้องการ {check})"
        return True, "valid"
    
    @classmethod
    def validate_credit_card(cls, number: str) -> Tuple[bool, str]:
        """ตรวจสอบ credit card ด้วย Luhn algorithm"""
        clean = re.sub(r'[\s-]', '', number)
        if not re.match(r'^\d{13,19}$', clean):
            return False, "รูปแบบไม่ถูกต้อง"
        card_type = None
        if cls.CREDIT_CARD_VISA.match(clean):         card_type = 'Visa'
        elif cls.CREDIT_CARD_MASTERCARD.match(clean): card_type = 'Mastercard'
        elif cls.CREDIT_CARD_AMEX.match(clean):       card_type = 'Amex'
        elif cls.CREDIT_CARD_JCB.match(clean):        card_type = 'JCB'
        # Luhn
        digits = [int(d) for d in reversed(clean)]
        total = sum(d if i % 2 == 0 else (d*2 - 9 if d*2 > 9 else d*2) for i, d in enumerate(digits))
        if total % 10 != 0:
            return False, "Luhn check ล้มเหลว"
        return True, card_type or "Unknown"
    
    @classmethod
    def validate_password(cls, password: str) -> Dict[str, Any]:
        checks = {
            'length_8':   len(password) >= 8,
            'length_12':  len(password) >= 12,
            'has_upper':  bool(re.search(r'[A-Z]', password)),
            'has_lower':  bool(re.search(r'[a-z]', password)),
            'has_digit':  bool(re.search(r'\d', password)),
            'has_special':bool(re.search(r'[!@#$%^&*()_+=\-\[\]{}|;:,.<>?]', password)),
            'no_spaces':  not bool(re.search(r'\s', password)),
            'no_repeats': not bool(re.search(r'(.)\1{2,}', password)),
        }
        score = sum(checks.values())
        strength = 'Strong' if score >= 7 else 'Medium' if score >= 5 else 'Weak' if score >= 3 else 'Very Weak'
        return {'checks': checks, 'score': score, 'strength': strength}
    
    @classmethod
    def validate_url(cls, url: str) -> Tuple[bool, Optional[Dict]]:
        m = cls.URL.match(url)
        if not m:
            return False, None
        port = m.group('port')
        if port and not (0 <= int(port) <= 65535):
            return False, None
        return True, {'scheme': m.group('scheme'), 'host': m.group('host'), 'port': port}


# ทดสอบ
v = Validator

print("Comprehensive Validation:")
print("=" * 60)

test_ids = ['1-1234-56789-01-2', '3-1004-00432-01-6']
print("\nThai National ID:")
for id_ in test_ids:
    valid, msg = v.validate_thai_id(id_)
    print(f"  {id_:<25} {'OK' if valid else 'FAIL'} {msg}")

test_cards = ['4532015112830366', '5425233430109903', '4532015112830367']
print("\nCredit Cards:")
for card in test_cards:
    valid, msg = v.validate_credit_card(card)
    masked = card[:4] + '****' + card[-4:]
    print(f"  {masked}  {'OK' if valid else 'FAIL'} {msg}")

passwords = ['abc', 'password123', 'P@ssw0rd!', 'MyStr0ng#Pass2024']
print("\nPassword Strength:")
for pw in passwords:
    result = v.validate_password(pw)
    print(f"  {pw:<20} {result['strength']:10} score={result['score']}/8")
```

---

## 36.2 Schema-based Validation

```python
import re
from typing import Any, Dict, List, Tuple

class SchemaValidator:
    """Validate objects against a JSON Schema-like definition"""
    
    TYPE_PATTERNS = {
        'email':    re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'),
        'url':      re.compile(r'^https?://[^\s]+$'),
        'phone_th': re.compile(r'^(?:\+?66|0)[6-9]\d{8}$'),
        'thai_id':  re.compile(r'^\d-\d{4}-\d{5}-\d{2}-\d$'),
        'date':     re.compile(r'^\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])$'),
        'slug':     re.compile(r'^[a-z0-9]+(?:-[a-z0-9]+)*$'),
        'uuid':     re.compile(r'^[0-9a-f]{8}-(?:[0-9a-f]{4}-){3}[0-9a-f]{12}$', re.I),
        'color_hex':re.compile(r'^#(?:[0-9a-fA-F]{3}){1,2}$'),
        'semver':   re.compile(r'^\d+\.\d+\.\d+(?:-[\w.]+)?(?:\+[\w.]+)?$'),
    }
    
    @classmethod
    def validate(cls, data: Dict, schema: Dict) -> Tuple[bool, List[str]]:
        errors = []
        for field, rules in schema.items():
            value = data.get(field)
            if rules.get('required', False) and value is None:
                errors.append(f"'{field}': required field missing")
                continue
            if value is None:
                continue
            ftype = rules.get('type')
            if ftype == 'string' and not isinstance(value, str):
                errors.append(f"'{field}': must be string")
            elif ftype == 'number' and not isinstance(value, (int, float)):
                errors.append(f"'{field}': must be number")
            fmt = rules.get('format')
            if fmt and isinstance(value, str):
                pattern = cls.TYPE_PATTERNS.get(fmt)
                if pattern and not pattern.match(str(value)):
                    errors.append(f"'{field}': invalid {fmt} format")
            pat = rules.get('pattern')
            if pat and isinstance(value, str):
                if not re.match(pat, value):
                    errors.append(f"'{field}': does not match pattern")
            if isinstance(value, (int, float)):
                if 'min' in rules and value < rules['min']:
                    errors.append(f"'{field}': must be >= {rules['min']}")
                if 'max' in rules and value > rules['max']:
                    errors.append(f"'{field}': must be <= {rules['max']}")
            if isinstance(value, str):
                if 'min_length' in rules and len(value) < rules['min_length']:
                    errors.append(f"'{field}': too short (min {rules['min_length']})")
                if 'max_length' in rules and len(value) > rules['max_length']:
                    errors.append(f"'{field}': too long (max {rules['max_length']})")
            if 'enum' in rules and value not in rules['enum']:
                errors.append(f"'{field}': must be one of {rules['enum']}")
        return len(errors) == 0, errors


USER_SCHEMA = {
    'username': {'required': True,  'type': 'string', 'min_length': 3, 'max_length': 30, 'pattern': r'^[a-zA-Z][a-zA-Z0-9_.-]{2,}$'},
    'email':    {'required': True,  'type': 'string', 'format': 'email'},
    'password': {'required': True,  'type': 'string', 'min_length': 8, 'max_length': 100},
    'phone':    {'required': False, 'type': 'string', 'format': 'phone_th'},
    'age':      {'required': False, 'type': 'number', 'min': 13, 'max': 120},
    'role':     {'required': False, 'type': 'string', 'enum': ['user', 'admin', 'moderator']},
}

test_users = [
    {'username': 'alice_123', 'email': 'alice@example.com', 'password': 'SecurePass123!', 'phone': '0812345678', 'age': 25, 'role': 'user'},
    {'username': 'b', 'email': 'not-an-email', 'password': 'abc', 'age': 10, 'role': 'superadmin'},
    {'username': 'charlie', 'password': 'mypassword'},
]

sv = SchemaValidator
print("\nSchema Validation:")
print("=" * 60)
for i, user in enumerate(test_users, 1):
    valid, errors = sv.validate(user, USER_SCHEMA)
    print(f"\nUser {i} [{'VALID' if valid else 'INVALID'}]:")
    if errors:
        for err in errors:
            print(f"  - {err}")
    else:
        print("  All fields valid!")
```

---

## 36.3 สรุป Part 36

```
Validation Patterns:
Thai ID:    ^\d-\d{4}-\d{5}-\d{2}-\d$  + checksum (sum * weight mod 11)
Visa:       ^4\d{12}(?:\d{3})?$
Mastercard: ^5[1-5]\d{14}$
Amex:       ^3[47]\d{13}$
Swift:      ^[A-Z]{6}[A-Z0-9]{2}(?:[A-Z0-9]{3})?$
UUID v4:    ^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$
Slug:       ^[a-z0-9]+(?:-[a-z0-9]+)*$
Semver:     ^\d+\.\d+\.\d+(?:-[\w.]+)?(?:\+[\w.]+)?$

Password strength:
- length >= 8/12
- [A-Z] uppercase, [a-z] lowercase
- \d digit, [!@#$] special
- no (.){3,} repeats, no spaces

Schema validation flow:
1. required check
2. type check (string/number/bool)
3. format check (email/url/phone)
4. pattern check (custom regex)
5. range/length check
6. enum check
```

---

*[← Part 35: Text Processing](part-35-text-processing.md) | [→ Part 37: Regex in Frameworks](part-37-frameworks.md)*
