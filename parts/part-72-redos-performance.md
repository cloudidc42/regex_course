# Part 72: ReDoS Prevention & Regex Performance

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-71

---

## 72.1 Understanding ReDoS (Regular Expression DoS)

```python
import re
import time
from typing import Callable, Tuple

print("ReDoS Prevention & Regex Performance:")
print("=" * 60)

print("\n1. Vulnerable vs safe patterns:")

def time_match(pattern_str: str, text: str, timeout: float = 1.0) -> Tuple[bool, float]:
    pattern = re.compile(pattern_str)
    start   = time.perf_counter()
    try:
        m = pattern.match(text)
        elapsed = time.perf_counter() - start
        return (m is not None, elapsed)
    except Exception as e:
        elapsed = time.perf_counter() - start
        return (False, elapsed)


# ReDoS-vulnerable patterns (exponential backtracking)
VULNERABLE_PATTERNS = [
    # Nested quantifiers - catastrophic on non-matching input
    (r'^(a+)+$',          'Nested quantifier (a+)+'),
    (r'^(a*)*$',          'Nested quantifier (a*)*'),
    (r'^(a|a)*$',         'Alternation ambiguity (a|a)*'),
    (r'^(a+|b+)*$',       'Alternation with quant (a+|b+)*'),
    # Overlapping groups
    (r'^(\w+\s+)+$',      'Overlapping groups (\\w+\\s+)+'),
    (r'^(\d+\.?)+$',      'Overlapping groups (\\d+\\.?)+'),
]

# Safe alternatives
SAFE_PATTERNS = [
    (r'^a+$',             'Simple quantifier a+'),
    (r'^a*$',             'Simple quantifier a*'),
    (r'^[ab]+$',          'Character class [ab]+'),
    (r'^(?:a+|b+)+$',     'Non-capturing (with care)'),
    (r'^\w+(?:\s+\w+)*$', 'Anchored with specific structure'),
    (r'^\d+(?:\.\d+)?$',  'Specific optional group'),
]

print(f"\n   Testing with non-matching input (short = safe, long = dangerous):")
print(f"\n   {'Pattern description':<35} {'Result':<8} {'Time (ms)'}")
print(f"   {'-'*35} {'-'*8} {'-'*10}")

# Use a moderately long non-matching string
test_input = 'a' * 20 + 'b'

for pattern_str, desc in VULNERABLE_PATTERNS[:3]:
    matched, elapsed = time_match(pattern_str, test_input)
    ms = elapsed * 1000
    status = 'match' if matched else 'no match'
    warn   = ' ⚠ SLOW' if ms > 10 else ''
    print(f"   {desc:<35} {status:<8} {ms:.2f}ms{warn}")

print()
for pattern_str, desc in SAFE_PATTERNS[:3]:
    matched, elapsed = time_match(pattern_str, test_input)
    ms = elapsed * 1000
    status = 'match' if matched else 'no match'
    print(f"   {desc:<35} {status:<8} {ms:.4f}ms")
```

---

## 72.2 ReDoS Prevention Techniques

```python
import re

print("\nReDoS Prevention Techniques:")
print("=" * 60)

print("\n1. Using atomic groups (regex module):")

try:
    import regex

    # Standard re: vulnerable (a+)+ allows backtracking into the group
    VULN = re.compile(r'^(a+)+$')

    # regex module: atomic group (?>a+) prevents backtracking into it
    SAFE_ATOMIC = regex.compile(r'^(?>a+)+$')

    # Possessive quantifier: a++ never gives back matched chars
    SAFE_POSS = regex.compile(r'^a++$')

    test_cases = [
        'aaa',
        'aaab',
        'a' * 10 + 'b',
    ]

    print(f"\n   {'Input':<20} {'Vuln re':<10} {'Atomic regex':<15} {'Possessive'}")
    print(f"   {'-'*20} {'-'*10} {'-'*15} {'-'*10}")
    for text in test_cases:
        mv = bool(VULN.match(text))
        ma = bool(SAFE_ATOMIC.match(text))
        mp = bool(SAFE_POSS.match(text))
        print(f"   {text!r:<20} {str(mv):<10} {str(ma):<15} {str(mp)}")

except ImportError:
    print("\n   (regex module not available)")
    print("   Workarounds without atomic groups:")
    print("   1. Use possessive-like rewrites")
    print("   2. Use anchors to limit input")
    print("   3. Validate length before matching")
    print("   4. Use specific character classes")


print("\n\n2. Safe email pattern (no backtracking catastrophe):")

UNSAFE_EMAIL = re.compile(r'^([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})*$')

SAFE_EMAIL = re.compile(
    r'^[a-zA-Z0-9._%+\-]{1,64}'
    r'@'
    r'[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?'
    r'(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?)*'
    r'\.[a-zA-Z]{2,10}$'
)

emails = [
    'user@example.com',
    'alice.smith+tag@sub.domain.co.uk',
    'invalid-email-no-at-sign',
    'user@',
    '@domain.com',
]

print(f"\n   {'Email':<40} {'Safe pattern'}")
print(f"   {'-'*40} {'-'*15}")
for email in emails:
    ok = bool(SAFE_EMAIL.match(email))
    print(f"   {email:<40} {'✓ valid' if ok else '✗ invalid'}")


print("\n\n3. Input length limits before regex:")

MAX_INPUT_LEN = {
    'email':    254,
    'username': 64,
    'url':      2048,
    'password': 128,
    'search':   500,
}


def safe_validate(field_type: str, value: str, pattern: re.Pattern) -> Tuple[bool, str]:
    max_len = MAX_INPUT_LEN.get(field_type, 1000)
    if len(value) > max_len:
        return False, f"Input too long: {len(value)} > {max_len}"
    return bool(pattern.match(value)), "ok"


print(f"\n   Length-guarded validation:")
test_vals = [
    ('email',    'user@example.com'),
    ('email',    'a' * 300 + '@example.com'),
    ('username', 'alice_123'),
    ('username', 'a' * 100),
]
for field, val in test_vals:
    ok, msg = safe_validate(field, val, SAFE_EMAIL if field == 'email' else re.compile(r'^\w+$'))
    print(f"   {field:<12} {val[:30]!r:<35} {'✓' if ok else '✗'} {msg}")
```

---

## 72.3 Regex Compilation & Caching

```python
import re
import time
from functools import lru_cache
from typing import Callable

print("\nRegex Compilation & Caching:")
print("=" * 60)

print("\n1. Pre-compiled vs dynamic compilation:")

def benchmark(name: str, fn: Callable, iterations: int = 1000) -> float:
    start = time.perf_counter()
    for _ in range(iterations):
        fn()
    elapsed = (time.perf_counter() - start) / iterations * 1_000_000
    print(f"   {name:<35} {elapsed:.2f} µs/call")
    return elapsed

EMAIL_COMPILED = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
TEST_EMAIL = 'alice@example.com'

t1 = benchmark("re.match (dynamic compile)",
    lambda: re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', TEST_EMAIL))

t2 = benchmark("compiled.match",
    lambda: EMAIL_COMPILED.match(TEST_EMAIL))

print(f"\n   Speedup from pre-compilation: {t1/t2:.1f}x")


print("\n\n2. Cached pattern compilation:")

@lru_cache(maxsize=256)
def get_pattern(pattern_str: str, flags: int = 0) -> re.Pattern:
    return re.compile(pattern_str, flags)


patterns_used = [
    (r'\b\d{4}-\d{2}-\d{2}\b', 0),
    (r'\b[A-Z]{2}\d{4}\b', 0),
    (r'\bemail\b', re.IGNORECASE),
]

for pat_str, flags in patterns_used:
    p = get_pattern(pat_str, flags)
    print(f"   Cached: {pat_str!r}")

cache_info = get_pattern.cache_info()
print(f"\n   Cache hits: {cache_info.hits}, misses: {cache_info.misses}")


print("\n\n3. finditer vs findall for large texts:")

large_text = 'email: test@example.com\n' * 10000
EMAIL_SIMPLE = re.compile(r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}')

t_all = benchmark("findall  (builds list)",
    lambda: EMAIL_SIMPLE.findall(large_text), iterations=10)

def count_finditer():
    count = 0
    for _ in EMAIL_SIMPLE.finditer(large_text):
        count += 1
    return count

t_iter = benchmark("finditer (lazy)",
    count_finditer, iterations=10)
```

---

## 72.4 Profiling & Optimization

```python
import re
import time
from typing import List

print("\nProfiling & Pattern Optimization:")
print("=" * 60)

print("\n1. Alternation order optimization:")

LOG_LEVELS_WORST = re.compile(r'\b(CRITICAL|ERROR|WARNING|INFO|DEBUG)\b')
LOG_LEVELS_BEST  = re.compile(r'\b(INFO|DEBUG|WARNING|ERROR|CRITICAL)\b')

log_sample = ' '.join(
    ['INFO: ok'] * 500 + ['DEBUG: cache'] * 300 + ['WARNING: slow'] * 100 +
    ['ERROR: failed'] * 50 + ['CRITICAL: down'] * 10
)

def run_n(pattern: re.Pattern, text: str, n: int = 5) -> float:
    start = time.perf_counter()
    for _ in range(n):
        pattern.findall(text)
    return (time.perf_counter() - start) / n * 1000

t_worst = run_n(LOG_LEVELS_WORST, log_sample)
t_best  = run_n(LOG_LEVELS_BEST, log_sample)

print(f"\n   CRITICAL first (rare first):  {t_worst:.2f}ms")
print(f"   INFO first (common first):    {t_best:.2f}ms")
print(f"   Improvement: {(t_worst - t_best) / t_worst * 100:.0f}%")


print("\n\n2. Specific vs broad character class:")

BROAD     = re.compile(r'\w+')
SPECIFIC  = re.compile(r'[a-z]+')
ALPHA_NUM = re.compile(r'[a-zA-Z0-9]+')

alpha_text = 'hello world python regex testing performance benchmark'

times = {}
for name, pat in [('\\w+ (broad)', BROAD), ('[a-z]+ (specific)', SPECIFIC)]:
    t = run_n(pat, alpha_text, 20)
    times[name] = t
    print(f"   {name:<30} {t:.3f}ms")


print("\n\n3. Anchor usage:")

NO_ANCHOR = re.compile(r'[A-Z]{3}\d{4}')
ANCHORED  = re.compile(r'^[A-Z]{3}\d{4}$')

test_ids = ['ABC1234', 'xyz9999', 'DEF5678', 'invalid123'] * 500

def check_ids(pattern: re.Pattern, ids: List[str]) -> int:
    return sum(1 for i in ids if pattern.match(i))

t1 = run_n(NO_ANCHOR, 'ABC1234', 100)
t2 = run_n(ANCHORED,  'ABC1234', 100)
print(f"\n   Unanchored:  {t1:.3f}ms")
print(f"   Anchored ^$: {t2:.3f}ms")
print(f"   Note: anchors prevent scanning full string unnecessarily")
```

---

## 72.5 สรุป Part 72

```
ReDoS Prevention & Performance:

1. Vulnerable patterns (avoid):
   (a+)+, (a*)*        → nested quantifiers
   (a|a)*              → ambiguous alternation
   (a+|b+)*            → quantified alternation
   (\w+\s+)+           → overlapping groups

2. Prevention techniques:
   a) Use atomic groups: (?>a+)  [regex module]
   b) Use possessive:    a++     [regex module]
   c) Rewrite without nested quantifiers
   d) Use anchors ^...$ to limit input
   e) Validate input length BEFORE regex

3. Performance rules:
   COMPILE once:  re.compile() outside loops
   CACHE:         use functools.lru_cache()
   FINDITER:      lazy, better than findall for large text
   ALTERNATION:   most-common option FIRST
   ANCHOR:        use ^...$ when possible
   CHAR CLASS:    [a-z] faster than \w when applicable

4. Length limits (prevent ReDoS via input):
   Email:    max 254 chars (RFC 5321)
   Username: max 64 chars
   URL:      max 2048 chars
   Search:   max 500 chars

5. Atomic group (regex module):
   (?>pattern) = match then never backtrack into
   a++          = possessive: same as (?>a+)
   Standard re: rewrite to avoid nested quantifiers
```

---

*[← Part 71: HTTP Security Patterns](part-71-http-smuggling.md) | [→ Part 73: Email & Communication Parsing](part-73-email-parsing.md)*
