# Part 93: Regex Testing & Quality Assurance

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~105 นาที | **ข้อกำหนด:** Part 01-92

---

## 93.1 Unit Testing Regex Patterns

```python
import re
import unittest
from typing import List, Tuple, Optional

print("Regex Testing & Quality Assurance:")
print("=" * 60)

print("\n1. Systematic regex unit testing:")

# Pattern under test: semantic version
SEMVER = re.compile(
    r'^(?P<major>0|[1-9]\d*)\.(?P<minor>0|[1-9]\d*)\.(?P<patch>0|[1-9]\d*)'
    r'(?:-(?P<prerelease>[0-9A-Za-z\-]+(?:\.[0-9A-Za-z\-]+)*))?'
    r'(?:\+(?P<buildmeta>[0-9A-Za-z\-]+(?:\.[0-9A-Za-z\-]+)*))?$'
)


class TestSemver(unittest.TestCase):
    """Test cases following the VALID / INVALID / GROUPS pattern."""

    VALID = [
        '0.0.0',
        '1.0.0',
        '1.2.3',
        '10.20.30',
        '1.0.0-alpha',
        '1.0.0-alpha.1',
        '1.0.0-0.3.7',
        '1.0.0+build.1',
        '1.0.0-beta+exp.sha.5114f85',
    ]

    INVALID = [
        '',
        '1',
        '1.2',
        '01.0.0',          # Leading zero
        '1.02.0',          # Leading zero
        '1.0.0.',          # Trailing dot
        '.1.0.0',          # Leading dot
        '1.0.0-',          # Trailing hyphen
        'a.b.c',           # Non-numeric
        '1.0.0 ',          # Trailing space
        '1.0.0\n',         # Newline
    ]

    GROUPS = [
        ('1.2.3',              {'major': '1', 'minor': '2', 'patch': '3', 'prerelease': None}),
        ('1.0.0-alpha.1',      {'major': '1', 'minor': '0', 'patch': '0', 'prerelease': 'alpha.1'}),
        ('2.0.0+build.007',    {'major': '2', 'minor': '0', 'patch': '0', 'buildmeta': 'build.007'}),
    ]

    def test_valid_inputs(self):
        for v in self.VALID:
            with self.subTest(v=v):
                self.assertIsNotNone(SEMVER.match(v), f"Expected match for {v!r}")

    def test_invalid_inputs(self):
        for v in self.INVALID:
            with self.subTest(v=v):
                self.assertIsNone(SEMVER.match(v), f"Expected no match for {v!r}")

    def test_group_extraction(self):
        for version, expected_groups in self.GROUPS:
            with self.subTest(v=version):
                m = SEMVER.match(version)
                self.assertIsNotNone(m, f"No match for {version!r}")
                for group_name, expected_val in expected_groups.items():
                    actual = m.group(group_name)
                    self.assertEqual(actual, expected_val,
                        f"Group {group_name!r}: expected {expected_val!r}, got {actual!r}")


# Run the tests programmatically
print(f"\n   Running semver pattern tests:")
loader = unittest.TestLoader()
suite = loader.loadTestsFromTestCase(TestSemver)
runner = unittest.TextTestRunner(verbosity=0, stream=open('/dev/null', 'w'))
result = runner.run(suite)

passed = result.testsRun - len(result.failures) - len(result.errors)
print(f"   Tests run: {result.testsRun}")
print(f"   Passed:    {passed}")
print(f"   Failed:    {len(result.failures)}")
print(f"   Errors:    {len(result.errors)}")

if result.failures:
    for test, msg in result.failures:
        print(f"   FAIL: {test}")
        print(f"     {msg[:100]}")
```

---

## 93.2 Property-Based Testing Concepts

```python
import re
import random
import string
from typing import List, Callable

print("\nProperty-Based Testing for Regex:")
print("=" * 60)

# Property: if a pattern matches, extracting the match and re-matching should still match
def property_match_extract_rematch(pattern: re.Pattern, text: str) -> bool:
    """Property: match → extract → re-match the extracted string."""
    m = pattern.search(text)
    if not m:
        return True  # No match is OK (vacuously true)
    extracted = m.group(0)
    return bool(pattern.match(extracted))


# Property: findall count matches finditer count
def property_findall_finditer_consistent(pattern: re.Pattern, text: str) -> bool:
    findall_count  = len(pattern.findall(text))
    finditer_count = sum(1 for _ in pattern.finditer(text))
    return findall_count == finditer_count


# Boundary condition generator
def boundary_strings(pattern_name: str) -> List[str]:
    """Generate boundary test cases."""
    boundaries = {
        'empty':         ['', ' ', '\n', '\t'],
        'unicode':       ['résumé', 'naïve', '日本語', '中文', '한국어', 'العربية'],
        'long':          ['a' * 1000, 'b' * 10000],
        'special_chars': ['\\n', '\\t', '\x00', '\xff', '\\\\', '"', "'"],
        'inject':        ['$(id)', '`whoami`', '; DROP TABLE', '<script>', '../../../'],
        'whitespace':    [' test ', '\ttest\t', '\ntest\n', 'test  test'],
    }
    return boundaries.get(pattern_name, [])


# Test EMAIL pattern against boundary conditions
EMAIL = re.compile(r'^[a-zA-Z0-9._%+\-]{1,64}@[a-zA-Z0-9.\-]{1,253}\.[a-zA-Z]{2,63}$')

print(f"\n   Boundary testing EMAIL pattern:")
boundary_categories = ['empty', 'unicode', 'special_chars', 'inject']
for category in boundary_categories:
    cases = boundary_strings(category)
    results = [(c, bool(EMAIL.match(c))) for c in cases]
    matches = [c for c, r in results if r]
    print(f"\n   [{category.upper()}] {len(cases)} cases, {len(matches)} matched:")
    for case, matched in results[:4]:
        status = 'MATCH' if matched else '----'
        print(f"     [{status}] {case!r}")


# Fuzz testing
def fuzz_pattern(pattern: re.Pattern, n: int = 50) -> List[str]:
    """Generate random strings and collect any that cause unexpected matches."""
    chars = string.printable
    unexpected = []
    for _ in range(n):
        length = random.randint(0, 50)
        rand_str = ''.join(random.choice(chars) for _ in range(length))
        try:
            result = pattern.match(rand_str)
            if result:
                unexpected.append(rand_str)
        except re.error as e:
            unexpected.append(f"ERROR: {e}")
    return unexpected

random.seed(42)
unexpected = fuzz_pattern(EMAIL, n=100)
print(f"\n   Fuzz test EMAIL (100 random inputs):")
print(f"   Unexpected matches: {len(unexpected)}")
for u in unexpected[:3]:
    print(f"     {u!r}")
```

---

## 93.3 Test Coverage & Equivalence Classes

```python
import re
from typing import List, Dict, Tuple

print("\nEquivalence Partitioning for Regex Tests:")
print("=" * 60)

def create_test_matrix(pattern_name: str, pattern: re.Pattern) -> Dict:
    if pattern_name == 'ipv4':
        return {
            'VALID_CLASS': {
                'description': 'Valid IPv4 addresses',
                'cases': [
                    ('0.0.0.0',         True,  'all zeros'),
                    ('255.255.255.255',  True,  'max valid'),
                    ('192.168.1.100',    True,  'typical private'),
                    ('10.0.0.1',         True,  'class A private'),
                    ('127.0.0.1',        True,  'loopback'),
                ],
            },
            'BOUNDARY_CLASS': {
                'description': 'Boundary values',
                'cases': [
                    ('0.0.0.1',          True,  'min last octet > 0'),
                    ('255.0.0.0',        True,  'max first octet'),
                    ('1.1.1.1',          True,  'all ones'),
                ],
            },
            'INVALID_CLASS': {
                'description': 'Invalid inputs',
                'cases': [
                    ('256.0.0.0',        False, 'octet > 255'),
                    ('192.168.1',        False, 'missing octet'),
                    ('192.168.1.1.1',    False, 'extra octet'),
                    ('192.168.01.1',     False, 'leading zero'),
                    ('',                 False, 'empty string'),
                    ('abc.def.ghi.jkl',  False, 'non-numeric'),
                ],
            },
        }
    return {}


IPV4_SIMPLE = re.compile(r'^(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})$')

def validate_ipv4(ip: str) -> bool:
    m = IPV4_SIMPLE.match(ip)
    if not m:
        return False
    return all(0 <= int(g) <= 255 for g in m.groups())


matrix = create_test_matrix('ipv4', IPV4_SIMPLE)
total_pass = 0
total_fail = 0

print(f"\n   IPv4 test matrix:")
for class_name, class_data in matrix.items():
    print(f"\n   [{class_name}] {class_data['description']}:")
    for ip, expected, desc in class_data['cases']:
        actual = validate_ipv4(ip)
        status = 'PASS' if actual == expected else 'FAIL'
        if status == 'PASS':
            total_pass += 1
        else:
            total_fail += 1
        icon = 'OK' if status == 'PASS' else 'XX'
        print(f"     [{icon}] {ip!r:<22} expected={expected}, got={actual} ({desc})")

print(f"\n   Results: {total_pass} passed, {total_fail} failed")
```

---

## 93.4 สรุป Part 93

```
Regex Testing & Quality Assurance:

1. Test structure (AAA pattern):
   Arrange: define VALID, INVALID, and GROUPS test cases
   Act: run pattern.match() / pattern.search() / pattern.findall()
   Assert: check match/no-match + group values

2. Test categories:
   Valid inputs: representative valid examples from each equivalence class
   Invalid inputs: near-misses (leading zero, trailing char, wrong format)
   Group extraction: verify named/positional group values
   Edge cases: empty string, Unicode, very long inputs

3. Equivalence partitioning:
   Group inputs with same expected behavior
   Boundary class: values at edges (min/max valid)
   Invalid class: values just outside valid range
   Test at least one case per class

4. Property-based testing:
   match → extract → re-match: result must still match
   findall count == finditer count
   Fuzz with random inputs: look for unexpected matches or exceptions

5. Regression testing:
   Keep a test file alongside every regex module
   Add tests when bugs are found (before fixing)
   Run tests in CI/CD pipeline
   Track match count changes across versions
```

---

*[← Part 92: Regex in Data Science & ML](part-92-data-science.md) | [→ Part 94: Regex in DevOps & Infrastructure](part-94-devops.md)*
