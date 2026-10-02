# Part 41: Testing Regex — การทดสอบ Pattern อย่างเป็นระบบ

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~65 นาที | **ข้อกำหนด:** Part 01-40

---

## 41.1 Unit Testing Regex Patterns

```python
import re
import unittest
from typing import Optional, Dict


class EmailValidator:
    PATTERN = re.compile(
        r'^[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~.-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    )
    
    @classmethod
    def is_valid(cls, email: str) -> bool:
        return bool(cls.PATTERN.match(email.strip()))
    
    @classmethod
    def extract_parts(cls, email: str) -> Optional[Dict]:
        m = re.match(r'^(?P<local>[^@]+)@(?P<domain>[^@]+)$', email)
        if m:
            parts = m.groupdict()
            domain_parts = parts['domain'].rsplit('.', 1)
            parts['tld'] = domain_parts[-1] if len(domain_parts) > 1 else ''
            return parts
        return None


class PhoneValidator:
    PATTERN_TH = re.compile(r'^(?:\+66|0)[6-9]\d{8}$')
    
    @classmethod
    def normalize(cls, phone: str) -> str:
        clean = re.sub(r'[\s\-\(\)]', '', phone)
        if clean.startswith('+66'):
            clean = '0' + clean[3:]
        return clean
    
    @classmethod
    def is_valid(cls, phone: str) -> bool:
        return bool(cls.PATTERN_TH.match(cls.normalize(phone)))


class TestEmailValidator(unittest.TestCase):
    
    def test_valid_emails(self):
        valid = [
            'user@example.com',
            'user.name@example.com',
            'user+tag@example.org',
            'user123@sub.domain.co.th',
            'a@b.io',
            'test_user@company-name.com',
        ]
        for email in valid:
            with self.subTest(email=email):
                self.assertTrue(EmailValidator.is_valid(email), f"Expected VALID: {email!r}")
    
    def test_invalid_emails(self):
        invalid = [
            '',
            'notanemail',
            '@example.com',
            'user@',
            'user @example.com',
            'user@example',
            'user@@example.com',
        ]
        for email in invalid:
            with self.subTest(email=email):
                self.assertFalse(EmailValidator.is_valid(email), f"Expected INVALID: {email!r}")
    
    def test_extract_parts(self):
        parts = EmailValidator.extract_parts('alice@example.com')
        self.assertIsNotNone(parts)
        self.assertEqual(parts['local'], 'alice')
        self.assertEqual(parts['domain'], 'example.com')
        self.assertEqual(parts['tld'], 'com')
    
    def test_case_sensitivity(self):
        self.assertTrue(EmailValidator.is_valid('USER@EXAMPLE.COM'))
        self.assertTrue(EmailValidator.is_valid('User@Example.Com'))


class TestPhoneValidator(unittest.TestCase):
    
    def test_valid_phones(self):
        valid = ['0812345678', '0912345678', '+66812345678', '0-81-234-5678', '081 234 5678']
        for phone in valid:
            with self.subTest(phone=phone):
                self.assertTrue(PhoneValidator.is_valid(phone), f"Expected VALID: {phone!r}")
    
    def test_invalid_phones(self):
        invalid = ['012345678', '08123456', '081234567890', 'abc', '']
        for phone in invalid:
            with self.subTest(phone=phone):
                self.assertFalse(PhoneValidator.is_valid(phone), f"Expected INVALID: {phone!r}")
    
    def test_normalization(self):
        tests = [
            ('+66812345678', '0812345678'),
            ('081-234-5678', '0812345678'),
            ('081 234 5678', '0812345678'),
        ]
        for raw, expected in tests:
            with self.subTest(raw=raw):
                self.assertEqual(PhoneValidator.normalize(raw), expected)


print("Running Unit Tests:")
print("=" * 60)
loader = unittest.TestLoader()
suite  = unittest.TestSuite()
suite.addTests(loader.loadTestsFromTestCase(TestEmailValidator))
suite.addTests(loader.loadTestsFromTestCase(TestPhoneValidator))
runner = unittest.TextTestRunner(verbosity=2)
result = runner.run(suite)
print(f"\nResults: {result.testsRun} tests, {len(result.failures)} failures, {len(result.errors)} errors")
```

---

## 41.2 Property-based Testing

```python
import re
import random
import string
from typing import Callable

class PropertyTest:
    def __init__(self, n: int = 100):
        self.n = n
    
    def check(self, property_fn: Callable[[], bool], name: str):
        failures = []
        for i in range(self.n):
            try:
                if not property_fn():
                    failures.append(i)
            except Exception as e:
                failures.append(f"error: {e}")
        status = "PASS" if not failures else f"FAIL (first: {failures[0]})"
        print(f"  Property '{name}': {status} ({self.n - len(failures)}/{self.n})")
        return not failures
    
    @staticmethod
    def random_email() -> str:
        chars = string.ascii_lowercase + string.digits + '._'
        local = ''.join(random.choices(chars, k=random.randint(3, 12)))
        domain = ''.join(random.choices(string.ascii_lowercase, k=random.randint(3, 8)))
        tld = random.choice(['com', 'org', 'net', 'io', 'co.th'])
        return f"{local}@{domain}.{tld}"
    
    @staticmethod
    def random_thai_phone() -> str:
        prefix = random.choice(['06', '07', '08', '09'])
        suffix = ''.join(random.choices(string.digits, k=8))
        return f"0{prefix[1]}{suffix}"


EMAIL_RE = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
PHONE_RE = re.compile(r'^0[6-9]\d{8}$')

tester = PropertyTest(n=200)

print("\nProperty-based Tests:")
print("=" * 60)

tester.check(
    lambda: EMAIL_RE.match(PropertyTest.random_email()) is not None,
    "random valid email matches pattern"
)
tester.check(
    lambda: PHONE_RE.match(PropertyTest.random_thai_phone()) is not None,
    "random valid phone matches pattern"
)
tester.check(
    lambda: EMAIL_RE.match('') is None,
    "empty string doesn't match email"
)

def check_at_parts():
    email = PropertyTest.random_email()
    if EMAIL_RE.match(email):
        parts = email.split('@')
        return len(parts) == 2 and all(parts)
    return True

tester.check(check_at_parts, "matched email has exactly one @")

tester.check(
    lambda: EMAIL_RE.match(PropertyTest.random_email() + ' extra') is None,
    "space injection breaks email match"
)
```

---

## 41.3 Test Table Pattern

```python
import re
from typing import Optional

def run_test_table(pattern: re.Pattern, tests: list, name: str):
    print(f"\n  Pattern: {pattern.pattern!r}")
    print(f"  {'Input':<28} {'Exp':<6} {'Got':<6} {'Pass':<5} Note")
    print(f"  {'-'*28} {'-'*6} {'-'*6} {'-'*5} ----")
    
    passed = 0
    for tc in tests:
        inp, expected, desc = tc[0], tc[1], tc[2]
        expected_groups = tc[3] if len(tc) > 3 else None
        
        m = pattern.match(inp)
        matched = m is not None
        ok = matched == expected
        
        if ok and expected_groups and m:
            ok = all(m.groupdict().get(k) == v for k, v in expected_groups.items())
        
        status = "OK" if ok else "FAIL"
        print(f"  {inp:<28} {str(expected):<6} {str(matched):<6} {status:<5} {desc}")
        if ok:
            passed += 1
    
    print(f"\n  Result: {passed}/{len(tests)} passed")
    return passed == len(tests)


print("\nTest Table Pattern:")
print("=" * 60)

DATE_RE = re.compile(
    r'^(?P<year>\d{4})-(?P<month>0[1-9]|1[0-2])-(?P<day>0[1-9]|[12]\d|3[01])$'
)

date_tests = [
    ('2024-10-15', True,  'valid date',       {'year': '2024', 'month': '10', 'day': '15'}),
    ('2024-01-01', True,  'New Year',          None),
    ('2024-12-31', True,  'Last day of year',  None),
    ('2024-13-01', False, 'invalid month 13',  None),
    ('2024-00-01', False, 'invalid month 00',  None),
    ('2024-10-32', False, 'invalid day 32',    None),
    ('24-10-15',   False, 'short year',        None),
    ('2024/10/15', False, 'wrong separator',   None),
    ('',           False, 'empty',             None),
    ('2024-10',    False, 'incomplete',        None),
]

run_test_table(DATE_RE, date_tests, "Date Pattern")

VERSION_RE = re.compile(
    r'^(?P<major>\d+)\.(?P<minor>\d+)\.(?P<patch>\d+)(?:-(?P<pre>[a-z0-9.]+))?$'
)

version_tests = [
    ('1.0.0',       True,  'simple version'),
    ('2.15.3',      True,  'multi-digit'),
    ('1.0.0-beta',  True,  'pre-release'),
    ('1.0.0-rc.1',  True,  'rc with number'),
    ('1.0',         False, 'missing patch'),
    ('v1.0.0',      False, 'leading v'),
    ('1.0.0.0',     False, 'too many parts'),
]

run_test_table(VERSION_RE, version_tests, "Version Pattern")
```

---

## 41.4 สรุป Part 41

```
Testing Checklist:
Positive: simple valid, minimal, maximal, all variations, unicode
Negative: empty, near-miss, wrong format, too short/long, wrong chars
Boundary: min length, max length, min-1, max+1, single char
Security: ReDoS 'a'*100+'!', newlines, null bytes, very long input

Python test tools:
- unittest.TestCase + subTest() สำหรับ parametrized tests
- Property-based: generate valid inputs แล้วตรวจบ invariants
- Test table: (input, expected, description) tuples

Key patterns:
- test_valid_* and test_invalid_* แยก positive/negative
- subTest(email=email) แสดง input ที่ fail
- extract_parts เพื่อตรวจเนื้อหา groups
```

---

*[← Part 40: Performance](part-40-performance.md) | [→ Part 42: Real-world Projects](part-42-projects.md)*
