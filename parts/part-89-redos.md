# Part 89: Regex Performance & ReDoS Prevention

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~100 นาที | **ข้อกำหนด:** Part 01-88

---

## 89.1 Understanding ReDoS (Regex Denial of Service)

```python
import re
import time
from typing import Tuple

print("Regex Performance & ReDoS Prevention:")
print("=" * 60)

print("\n1. ReDoS vulnerability patterns:")

# --- VULNERABLE patterns (DO NOT use in production) ---
# These exhibit exponential or polynomial backtracking

# Pattern: nested quantifiers (catastrophic backtracking)
# Example: (a+)+ on input "aaaaaaaaab"
VULNERABLE_1 = r'(a+)+'           # Nested quantifier: VULNERABLE

# Pattern: alternation with overlap
# Example: (a|aa)+ on input "aaaaab"
VULNERABLE_2 = r'(a|aa)+'         # Overlapping alternation: VULNERABLE

# Pattern: anchor + wildcard + anchor
# Example: ^(a+)+$ on "aaaaaaaaab"
VULNERABLE_3 = r'^(a+)+$'         # Anchored nested: VERY VULNERABLE

# Classic email ReDoS (simplified)
VULNERABLE_EMAIL = r'^[a-zA-Z0-9.!#$%&\'*+/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$'
# ↑ Actually this one is fine, but the following version is not:
VULNERABLE_EMAIL_BAD = r'^([a-zA-Z0-9])(([\.\-]|[_]+)?([a-zA-Z0-9]+))*(@){1}[a-z0-9]+[.]{1}(([a-z]{2,3})|([a-z]{2,3}[.]{1}[a-z]{2,3}))$'


def measure_regex_time(pattern: str, test_string: str, timeout: float = 2.0) -> Tuple[float, bool]:
    """Measure regex match time, return (seconds, timed_out)."""
    compiled = re.compile(pattern)
    start = time.monotonic()
    try:
        compiled.match(test_string)
        elapsed = time.monotonic() - start
        return elapsed, False
    except Exception:
        elapsed = time.monotonic() - start
        return elapsed, True


# Safe inputs vs malicious ReDoS inputs
print(f"\n   ReDoS timing comparison (safe vs attack input):")
tests = [
    ('(a+)+',      'aaaaaaaaaa',         'aaaaaaaaaaaaaaaaab',     'nested quantifier'),
    ('(a|aa)+',    'aaaaaaaaaa',         'aaaaaaaaaaaaaaaaab',     'overlapping alternation'),
]

for pattern, safe_input, attack_input, desc in tests:
    t_safe,   _ = measure_regex_time(pattern, safe_input)
    t_attack, _ = measure_regex_time(pattern, attack_input[:20])  # Limit attack length for demo

    print(f"\n   Pattern: {pattern!r} ({desc})")
    print(f"   Safe input  {safe_input!r:<25} → {t_safe*1000:.2f}ms")
    print(f"   Attack input {attack_input[:20]!r:<25} → {t_attack*1000:.2f}ms")
    ratio = t_attack / t_safe if t_safe > 0 else float('inf')
    print(f"   Ratio: {ratio:.1f}x {'VULNERABLE' if ratio > 10 else 'OK'}")
```

---

## 89.2 Safe Pattern Replacements

```python
import re
from typing import Optional

print("\nSafe Pattern Replacements:")
print("=" * 60)

# --- EMAIL VALIDATION ---

# UNSAFE (ReDoS potential in some implementations)
UNSAFE_EMAIL = re.compile(
    r'^([a-zA-Z0-9])(([\.\-]|[_]+)?([a-zA-Z0-9]+))*(@)([a-z0-9]+)'
    r'[.](([a-z]{2,3})|([a-z]{2,3}[.][a-z]{2,3}))$'
)

# SAFE: Deterministic, no nested quantifiers
SAFE_EMAIL = re.compile(
    r'^[a-zA-Z0-9._%+\-]{1,64}@[a-zA-Z0-9.\-]{1,253}\.[a-zA-Z]{2,63}$'
)

# --- URL VALIDATION ---

# UNSAFE (overlapping character classes, nested quantifiers)
UNSAFE_URL = re.compile(
    r'^(https?://)?(([0-9a-zA-Z]([0-9a-zA-Z-]*[0-9a-zA-Z])*\.)+[0-9a-zA-Z]'
    r'[0-9a-zA-Z-]*[0-9a-zA-Z])(:\d+)?(/[^?#]*)?(\?[^#]*)?(#.*)?$'
)

# SAFE: Simple, linear components, possessive-style via atomic design
SAFE_URL = re.compile(
    r'^https?://[^\s/$.?#][^\s]*$'
)

# --- IP ADDRESS VALIDATION ---

SAFE_IP = re.compile(r'^(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})$')

def validate_ip_safe(ip: str) -> bool:
    m = SAFE_IP.match(ip)
    if not m:
        return False
    # Validate numeric ranges in Python, not regex
    return all(0 <= int(g) <= 255 for g in m.groups())


# --- PASSWORD VALIDATION ---

def validate_password_safe(password: str) -> dict:
    """Validate password constraints without nested quantifiers."""
    checks = {
        'length':    len(password) >= 8,
        'uppercase': bool(re.search(r'[A-Z]', password)),
        'lowercase': bool(re.search(r'[a-z]', password)),
        'digit':     bool(re.search(r'\d', password)),
        'special':   bool(re.search(r'[@$!%*?&]', password)),
        'no_spaces': not re.search(r'\s', password),
    }
    checks['valid'] = all(checks.values())
    return checks


test_emails = [
    'user@example.com',
    'user+tag@sub.domain.co.uk',
    'invalid@',
    '@nodomain.com',
    'a' * 65 + '@example.com',  # Too long local part
]

print(f"\n   Safe email validation:")
for email in test_emails:
    safe_result = bool(SAFE_EMAIL.match(email))
    print(f"   {'OK' if safe_result else 'FAIL'}: {email[:50]!r}")

print(f"\n   Safe IP validation:")
test_ips = ['192.168.1.1', '10.0.0.0', '256.0.0.1', '0.0.0.0', '255.255.255.255']
for ip in test_ips:
    result = validate_ip_safe(ip)
    print(f"   {'OK' if result else 'FAIL'}: {ip!r}")

print(f"\n   Safe password validation:")
passwords = ['Password1!', 'weakpass', 'NoSpecial1', 'Short1!', 'GoodP@ssw0rd']
for pwd in passwords:
    result = validate_password_safe(pwd)
    status = 'VALID' if result['valid'] else 'INVALID'
    print(f"   [{status}] {pwd!r}")
    if not result['valid']:
        failed = [k for k, v in result.items() if not v and k != 'valid']
        print(f"     failed: {failed}")
```

---

## 89.3 Regex Complexity Analysis

```python
import re
import time
from typing import Tuple

print("\nRegex Complexity Analysis:")
print("=" * 60)


def analyze_pattern_safety(pattern: str) -> dict:
    """Analyze a regex pattern for ReDoS risk."""
    risks = []

    # Check for nested quantifiers
    if re.search(r'\([^()]*(?:\*|\+|\?|\{[^}]+\})[^()]*\)(?:\*|\+|\?|\{[^}]+\})', pattern):
        risks.append('nested_quantifier')

    # Count alternations
    alt_count = pattern.count('|')
    if alt_count > 20:
        risks.append(f'excessive_alternation ({alt_count})')

    # Check for overlapping character classes in alternation
    if re.search(r'\[([^\]]+)\]\|(?:\[[^\]]*\]|.)+\[([^\]]+)\]', pattern):
        risks.append('possible_overlapping_alternation')

    # Check for atomic group absence with catastrophic potential
    if re.search(r'\([^?].*[+*]\).*[+*]', pattern):
        risks.append('potential_catastrophic_backtracking')

    # Measure against test input
    test_malicious = 'a' * 30 + 'b'
    try:
        compiled = re.compile(pattern)
        start = time.monotonic()
        compiled.match(test_malicious)
        elapsed = time.monotonic() - start
        if elapsed > 0.1:
            risks.append(f'slow_on_test_input ({elapsed*1000:.0f}ms)')
    except re.error as e:
        risks.append(f'invalid_pattern: {e}')

    return {
        'pattern': pattern[:60],
        'risks': risks,
        'safe': len(risks) == 0
    }


patterns_to_analyze = [
    r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,63}$',   # Safe email
    r'(a+)+',                                                        # Nested quantifier
    r'^(\w+\s?)+$',                                                  # Nested quantifier
    r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b',                    # Safe IP
    r'^(https?|ftp)://[^\s/$.?#][^\s]*$',                          # Safe URL
    r'(\w|\d)+',                                                     # Overlapping classes
]

print(f"\n   Pattern safety analysis:")
for p in patterns_to_analyze:
    result = analyze_pattern_safety(p)
    status = 'SAFE' if result['safe'] else 'RISKY'
    print(f"\n   [{status}] {p[:60]!r}")
    for risk in result['risks']:
        print(f"     → {risk}")
```

---

## 89.4 สรุป Part 89

```
Regex Performance & ReDoS Prevention:

1. What is ReDoS?
   Regex Denial of Service: crafted input forces exponential backtracking
   Severity: can hang server for seconds/minutes per request
   CVE examples: CVE-2019-20149 (kind-of), many npm packages

2. Vulnerable patterns to avoid:
   (a+)+     → nested quantifier: exponential backtracking
   (a|aa)+   → overlapping alternation: exponential
   ^(a+)+$   → anchored nested: WORST CASE (no short-circuit)
   (\w|\d)+  → overlapping character class alternation

3. Safe alternatives:
   Instead of (a+)+  → use a+ (remove outer group with quantifier)
   Instead of (a|aa)+ → use a{1,2}+ or rewrite with possessive (a++)
   Long patterns → validate length FIRST in Python before applying regex
   IP/email → use simple regex + Python range/domain validation

4. Python-specific mitigations:
   re module: no possessive quantifiers or atomic groups (unlike PCRE)
   timeout via threading: run match in thread, kill after N seconds
   Input length limits: reject strings > N chars before regex
   Use re2 library: linear-time guarantee (no backtracking)

5. Best practices:
   Benchmark patterns: use timeit with malicious input (aaa...b)
   Test both safe AND malicious inputs
   Static analysis: tools like safe-regex, vuln-regex-detector
   Simplify: simple patterns > complex patterns
   Separate concerns: parse first, validate second
```

---

*[← Part 88: WAF Evasion Techniques](part-88-waf-evasion.md) | [→ Part 90: Advanced Python re Module](part-90-advanced-re.md)*
