# Part 55: Regex Testing & Test-Driven Development

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-54

---

## 55.1 Unit Testing Regex Patterns

```python
import re
import unittest

print("Regex Testing with unittest:")
print("=" * 60)

print("\n1. Basic regex test structure:")

class TestEmailRegex(unittest.TestCase):
    """Test suite for email validation regex"""

    EMAIL_RE = re.compile(
        r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    )

    def test_simple_email(self):
        self.assertIsNotNone(self.EMAIL_RE.match('user@example.com'))

    def test_subdomain(self):
        self.assertIsNotNone(self.EMAIL_RE.match('user@mail.example.com'))

    def test_plus_tag(self):
        self.assertIsNotNone(self.EMAIL_RE.match('user+tag@example.com'))

    def test_dots_in_local(self):
        self.assertIsNotNone(self.EMAIL_RE.match('first.last@example.com'))

    def test_long_tld(self):
        self.assertIsNotNone(self.EMAIL_RE.match('user@example.travel'))

    def test_no_at_sign(self):
        self.assertIsNone(self.EMAIL_RE.match('userexample.com'))

    def test_no_domain(self):
        self.assertIsNone(self.EMAIL_RE.match('user@'))

    def test_no_tld(self):
        self.assertIsNone(self.EMAIL_RE.match('user@example'))

    def test_empty_string(self):
        self.assertIsNone(self.EMAIL_RE.match(''))

    def test_double_at(self):
        self.assertIsNone(self.EMAIL_RE.match('user@@example.com'))


suite = unittest.TestLoader().loadTestsFromTestCase(TestEmailRegex)
runner = unittest.TextTestRunner(verbosity=0, stream=__import__('io').StringIO())
result = runner.run(suite)
print(f"   TestEmailRegex: {result.testsRun} tests, "
      f"{len(result.failures)} failures, {len(result.errors)} errors")
if result.failures:
    for test, msg in result.failures:
        print(f"   FAIL: {test}")
else:
    print("   All tests passed ✓")
```

---

## 55.2 Parameterized Testing

```python
import re
import unittest

print("\nParameterized Test Cases:")
print("=" * 60)

class RegexTestCase(unittest.TestCase):
    """Base class for parameterized regex tests"""

    def assert_matches(self, pattern: re.Pattern, cases: list):
        for case in cases:
            with self.subTest(value=case):
                self.assertIsNotNone(
                    pattern.fullmatch(case),
                    f"Expected {case!r} to match {pattern.pattern!r}"
                )

    def assert_no_match(self, pattern: re.Pattern, cases: list):
        for case in cases:
            with self.subTest(value=case):
                self.assertIsNone(
                    pattern.fullmatch(case),
                    f"Expected {case!r} NOT to match {pattern.pattern!r}"
                )


class TestPhoneThailand(RegexTestCase):

    PHONE = re.compile(r'0[6-9]\d{8}')

    def test_valid_numbers(self):
        self.assert_matches(self.PHONE, [
            '0812345678',
            '0912345678',
            '0611111111',
            '0799999999',
        ])

    def test_invalid_numbers(self):
        self.assert_no_match(self.PHONE, [
            '1234567890',
            '0512345678',
            '081234567',
            '08123456789',
            '08-1234-5678',
            '',
        ])


class TestIPv4(RegexTestCase):

    IPv4 = re.compile(
        r'(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}'
        r'(?:25[0-5]|2[0-4]\d|[01]?\d\d?)'
    )

    def test_valid_ips(self):
        self.assert_matches(self.IPv4, [
            '0.0.0.0',
            '127.0.0.1',
            '192.168.1.1',
            '255.255.255.255',
            '10.0.0.1',
            '172.16.0.1',
        ])

    def test_invalid_ips(self):
        self.assert_no_match(self.IPv4, [
            '256.0.0.1',
            '192.168.1',
            '1.2.3.4.5',
            '192.168.1.1a',
            '',
        ])


for cls in [TestPhoneThailand, TestIPv4]:
    suite = unittest.TestLoader().loadTestsFromTestCase(cls)
    runner = unittest.TextTestRunner(verbosity=0, stream=__import__('io').StringIO())
    result = runner.run(suite)
    status = "✓ PASS" if result.wasSuccessful() else "✗ FAIL"
    print(f"\n   {cls.__name__}: {result.testsRun} tests — {status}")
    for test, msg in result.failures + result.errors:
        print(f"   FAIL: {test}: {msg[:80]}")
```

---

## 55.3 Property-Based Testing

```python
import re
import string
import random

print("\nProperty-Based Testing for Regex:")
print("=" * 60)

print("\n1. Generate test inputs from character classes:")

def random_string(chars: str, min_len: int, max_len: int) -> str:
    length = random.randint(min_len, max_len)
    return ''.join(random.choice(chars) for _ in range(length))

def run_property_test(label: str, pattern: re.Pattern,
                      generator, n: int = 200) -> None:
    failures = []
    for _ in range(n):
        value, should_match = generator()
        matched = bool(pattern.fullmatch(value))
        if matched != should_match:
            failures.append((value, should_match, matched))
    if failures:
        print(f"   {label}: {len(failures)}/{n} FAILED")
        for v, expected, got in failures[:3]:
            print(f"     {v!r}: expected={expected}, got={got}")
    else:
        print(f"   {label}: {n}/{n} passed ✓")


HEX_COLOR = re.compile(r'^#(?:[0-9a-fA-F]{3}){1,2}$')

def hex_color_generator():
    if random.random() < 0.5:
        hex_chars = '0123456789abcdefABCDEF'
        length = random.choice([3, 6])
        color = '#' + random_string(hex_chars, length, length)
        return color, True
    else:
        color = '#' + random_string(string.ascii_letters + string.digits, 1, 8)
        valid = bool(HEX_COLOR.match(color))
        return color, valid

run_property_test("HexColor", HEX_COLOR, hex_color_generator, 500)


DATE_ISO = re.compile(r'^\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])$')

def date_generator():
    if random.random() < 0.5:
        year  = random.randint(1900, 2099)
        month = random.randint(1, 12)
        day   = random.randint(1, 28)
        return f'{year:04d}-{month:02d}-{day:02d}', True
    else:
        year  = random.randint(1900, 2099)
        month = random.choice([0, 13, 99])
        day   = random.randint(1, 28)
        return f'{year:04d}-{month:02d}-{day:02d}', False

run_property_test("DateISO", DATE_ISO, date_generator, 500)


print("\n2. Boundary testing:")

SLUG = re.compile(r'^[a-z0-9]+(?:-[a-z0-9]+)*$')

boundary_cases = [
    ('a', True),
    ('a' * 100, True),
    ('a-b', True),
    ('a-b-c-d', True),
    ('-a', False),
    ('a-', False),
    ('a--b', False),
    ('A', False),
    ('a b', False),
    ('', False),
]

print(f"\n   {'Input':<20} {'Expected':<10} {'Got':<10} {'Status'}")
print(f"   {'-'*20} {'-'*10} {'-'*10} {'-'*8}")
for value, expected in boundary_cases:
    got = bool(SLUG.fullmatch(value))
    status = '✓' if got == expected else '✗ FAIL'
    display = repr(value) if len(value) < 12 else f'{repr(value[:10])}...'
    print(f"   {display:<20} {str(expected):<10} {str(got):<10} {status}")
```

---

## 55.4 Regex Debugging Utilities

```python
import re
from typing import Dict, List

print("\nRegex Debugging Utilities:")
print("=" * 60)

print("\n1. Match explainer:")

def explain_match(pattern: re.Pattern, text: str):
    m = pattern.match(text)
    if not m:
        print(f"   No match for {text!r}")
        return

    print(f"\n   Pattern: {pattern.pattern!r}")
    print(f"   Input:   {text!r}")
    print(f"   Full match: {m.group()!r}")

    if m.groupdict():
        print(f"   Named groups:")
        for name, value in m.groupdict().items():
            print(f"     {name:<15} = {value!r}")

    if m.groups():
        print(f"   Groups: {m.groups()}")
    print(f"   Span: {m.span()}")


LOG_RE = re.compile(
    r'(?P<date>\d{4}-\d{2}-\d{2})\s+'
    r'(?P<time>\d{2}:\d{2}:\d{2})\s+'
    r'(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\s+'
    r'(?P<message>.+)'
)

explain_match(LOG_RE, "2024-10-15 09:30:00 ERROR Connection failed")


print("\n\n2. Find all overlapping matches:")

def findall_overlapping(pattern: str, text: str) -> List[str]:
    results = []
    compiled = re.compile(f'(?=({pattern}))')
    for m in compiled.finditer(text):
        results.append(m.group(1))
    return results

text = "abcabc"
print(f"\n   Text: {text!r}")
normal  = re.findall(r'abc', text)
overlap = findall_overlapping(r'abc', text)
print(f"   Normal findall('abc'): {normal}")
print(f"   Overlapping:           {overlap}")

text2 = "aababc"
print(f"\n   Text: {text2!r}")
print(f"   Overlapping 'ab':  {findall_overlapping('ab', text2)}")


print("\n3. Pattern complexity analysis:")

def analyze_pattern(pattern: str) -> Dict:
    issues = []
    compiled = None
    try:
        compiled = re.compile(pattern)
    except re.error as e:
        return {'valid': False, 'error': str(e)}

    if re.search(r'\([^)]*[+*][^)]*\)[+*]', pattern):
        issues.append("Nested quantifiers detected — possible ReDoS risk")
    if re.search(r'\((?:[^)]+[+*]){2,}\)', pattern):
        issues.append("Multiple quantifiers inside group — check for ReDoS")

    alts = pattern.count('|')
    if alts > 10:
        issues.append(f"High alternation count ({alts}) — may be slow")

    if not (pattern.startswith('^') or pattern.endswith('$')):
        issues.append("No anchors — consider ^ and $ for validation")

    cap_groups   = len(re.findall(r'\((?!\?)', pattern))
    noncap_groups = len(re.findall(r'\(\?:', pattern))

    return {
        'valid':         True,
        'issues':        issues,
        'groups':        compiled.groups,
        'cap_groups':    cap_groups,
        'noncap_groups': noncap_groups,
        'named_groups':  list(compiled.groupindex.keys()),
    }

patterns_to_analyze = [
    r'\d+',
    r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
    r'(a+)+b',
    r'(?:INFO|WARNING|ERROR|DEBUG|CRITICAL|NOTICE|VERBOSE|TRACE)',
]

for p in patterns_to_analyze:
    info = analyze_pattern(p)
    print(f"\n   Pattern: {p!r}")
    if not info['valid']:
        print(f"   INVALID: {info['error']}")
    else:
        print(f"   Groups: {info['groups']} (cap: {info['cap_groups']}, non-cap: {info['noncap_groups']})")
        if info['named_groups']:
            print(f"   Named: {info['named_groups']}")
        for issue in info['issues']:
            print(f"   ⚠ {issue}")
        if not info['issues']:
            print(f"   ✓ No issues detected")
```

---

## 55.5 สรุป Part 55

```
Regex Testing Best Practices:

1. Test Structure:
   - Valid inputs:   should match (positive cases)
   - Invalid inputs: should NOT match (negative cases)
   - Edge cases: empty, min/max length, boundary values

2. unittest for Regex:
   assertIsNotNone(pattern.match(value))   — must match
   assertIsNone(pattern.match(value))      — must NOT match
   subTest(value=case)                     — show which case failed

3. Parameterized Tests:
   Use subTest() to run same assertion over many values
   Report exactly which value failed

4. Property-Based Testing:
   Generate many random valid inputs → all should match
   Generate many random invalid inputs → none should match
   Test boundary conditions explicitly

5. Debugging:
   - Print named groups on failure: m.groupdict()
   - Use re.DEBUG flag: re.compile(pattern, re.DEBUG)
   - Test pieces separately before combining

6. Common Pitfalls:
   match() vs fullmatch(): use fullmatch() for validation
   Greedy vs lazy: test with inputs that have many matches
   Anchors: ^ and $ must be present for validation
   re.IGNORECASE: test with mixed case inputs
```

---

*[← Part 54: Performance Optimization](part-54-performance.md) | [→ Part 56: String Processing Pipelines](part-56-string-pipelines.md)*
