# Part 95: Regex Performance Optimization

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~110 นาที | **ข้อกำหนด:** Part 01-94

---

## 95.1 Benchmarking Regex Performance

```python
import re
import timeit
from typing import List, Tuple, Callable

print("Regex Performance Optimization:")
print("=" * 60)

print("\n1. Benchmarking compiled vs non-compiled patterns:")

TEST_TEXT = """
2024-01-15 10:30:45 INFO User alice logged in from 192.168.1.100
2024-01-15 10:30:46 ERROR Database connection failed: timeout after 30s
2024-01-15 10:30:47 WARNING Rate limit exceeded for user bob from 10.0.0.5
2024-01-15 10:30:48 INFO User charlie logged out
""" * 100  # 400 lines

# Compiled vs non-compiled
COMPILED_IP = re.compile(r'\b\d{1,3}(?:\.\d{1,3}){3}\b')

def find_ips_compiled(text: str) -> List[str]:
    return COMPILED_IP.findall(text)

def find_ips_uncompiled(text: str) -> List[str]:
    return re.findall(r'\b\d{1,3}(?:\.\d{1,3}){3}\b', text)

N = 1000
t_compiled   = timeit.timeit(lambda: find_ips_compiled(TEST_TEXT),   number=N)
t_uncompiled = timeit.timeit(lambda: find_ips_uncompiled(TEST_TEXT), number=N)

print(f"\n   IP extraction ({N} iterations on 400-line text):")
print(f"   Compiled:   {t_compiled:.3f}s  ({t_compiled/N*1000:.3f}ms per call)")
print(f"   Uncompiled: {t_uncompiled:.3f}s  ({t_uncompiled/N*1000:.3f}ms per call)")
print(f"   Speedup:    {t_uncompiled/t_compiled:.1f}x")


print("\n2. Anchoring and early failure:")

UNANCHORED = re.compile(r'[A-Z]{3}-\d{4}')
ANCHORED   = re.compile(r'^[A-Z]{3}-\d{4}')

no_match_texts = ['lowercase-text', 'numbers 12345', 'mixed CASE text'] * 1000

t_unanchored = timeit.timeit(lambda: [UNANCHORED.match(t) for t in no_match_texts], number=100)
t_anchored   = timeit.timeit(lambda: [ANCHORED.match(t) for t in no_match_texts],   number=100)

print(f"\n   Non-matching strings (anchored vs unanchored):")
print(f"   Unanchored: {t_unanchored:.3f}s")
print(f"   Anchored:   {t_anchored:.3f}s")
print(f"   Speedup:    {t_unanchored/t_anchored:.1f}x")


print("\n3. Character class vs alternation:")

ALT_VOWEL   = re.compile(r'(a|e|i|o|u)', re.IGNORECASE)
CLASS_VOWEL = re.compile(r'[aeiouAEIOU]')

long_text = 'the quick brown fox jumps over the lazy dog ' * 500

t_alt   = timeit.timeit(lambda: ALT_VOWEL.findall(long_text),   number=500)
t_class = timeit.timeit(lambda: CLASS_VOWEL.findall(long_text), number=500)

print(f"\n   Vowel finding (500 iterations):")
print(f"   Alternation (a|e|i|o|u): {t_alt:.3f}s")
print(f"   Character class [aeiou]:  {t_class:.3f}s")
print(f"   Speedup: {t_alt/t_class:.1f}x")
```

---

## 95.2 Pattern Optimization Techniques

```python
import re
import timeit
from typing import List

print("\nPattern Optimization Techniques:")
print("=" * 60)

# Non-capturing groups (?:...) vs capturing (...)
CAPTURING     = re.compile(r'(https?)://([\w.\-]+)(/[\w/]*)?')
NON_CAPTURING = re.compile(r'(?:https?)://(?:[\w.\-]+)(?:/[\w/]*)?')


# Pre-filter before regex
def find_ips_naive(lines: List[str]) -> List[str]:
    IP = re.compile(r'\b(\d{1,3}(?:\.\d{1,3}){3})\b')
    return [m.group(1) for line in lines for m in IP.finditer(line)]

def find_ips_filtered(lines: List[str]) -> List[str]:
    """Only apply regex to lines that likely contain IPs."""
    IP = re.compile(r'\b(\d{1,3}(?:\.\d{1,3}){3})\b')
    results = []
    for line in lines:
        if '.' in line and any(c.isdigit() for c in line[:20]):
            results.extend(m.group(1) for m in IP.finditer(line))
    return results


log_lines = [
    f'2024-01-15 INFO User {"user"+str(i)} logged in from 192.168.1.{i%255}' if i % 3 == 0
    else f'2024-01-15 DEBUG Processing request number {i}'
    for i in range(500)
]

t_naive    = timeit.timeit(lambda: find_ips_naive(log_lines),    number=200)
t_filtered = timeit.timeit(lambda: find_ips_filtered(log_lines), number=200)

print(f"\n   Pre-filter optimization (200 iterations, 500 log lines):")
print(f"   Naive (regex all):    {t_naive:.3f}s")
print(f"   Pre-filtered:         {t_filtered:.3f}s")
print(f"   Speedup: {t_naive/t_filtered:.1f}x")

naive_result    = sorted(find_ips_naive(log_lines))
filtered_result = sorted(find_ips_filtered(log_lines))
print(f"   Results match: {naive_result == filtered_result}")


url_text = 'Visit https://example.com/path or http://test.org/page for more' * 100

t_cap     = timeit.timeit(lambda: CAPTURING.findall(url_text),     number=1000)
t_non_cap = timeit.timeit(lambda: NON_CAPTURING.findall(url_text), number=1000)
print(f"\n   Capturing vs non-capturing (1000 iterations):")
print(f"   Capturing:     {t_cap:.3f}s")
print(f"   Non-capturing: {t_non_cap:.3f}s")
```

---

## 95.3 Large-Scale Regex Processing

```python
import re
import timeit
from typing import Generator, List, Dict

print("\nLarge-Scale Regex Processing:")
print("=" * 60)

# Batch regex matching
def multi_pattern_match(text: str, patterns: Dict[str, re.Pattern]) -> Dict[str, List]:
    results = {name: [] for name in patterns}
    for name, pattern in patterns.items():
        results[name] = pattern.findall(text)
    return results


# Combined pattern - single pass
COMBINED_PATTERN = re.compile(
    r'(?P<ip>\b\d{1,3}(?:\.\d{1,3}){3}\b)|'
    r'(?P<email>[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})|'
    r'(?P<url>https?://[^\s]+)|'
    r'(?P<date>\d{4}-\d{2}-\d{2})'
)

PATTERNS_SEPARATE = {
    'ip':     re.compile(r'\b\d{1,3}(?:\.\d{1,3}){3}\b'),
    'email':  re.compile(r'[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}'),
    'url':    re.compile(r'https?://[^\s]+'),
    'date':   re.compile(r'\d{4}-\d{2}-\d{2}'),
}

def extract_all_combined(text: str) -> Dict[str, List[str]]:
    results = {'ip': [], 'email': [], 'url': [], 'date': []}
    for m in COMBINED_PATTERN.finditer(text):
        entity_type = m.lastgroup
        if entity_type:
            results[entity_type].append(m.group(0))
    return results


sample_text = """
Contact: alice@example.com or visit https://example.com/contact
Server IP: 192.168.1.100, Backup: 10.0.0.5
Incident date: 2024-01-15, resolved 2024-01-16
API endpoint: https://api.example.com/v1/users
Admin: admin@company.org
""" * 20

t_separate = timeit.timeit(lambda: multi_pattern_match(sample_text, PATTERNS_SEPARATE), number=500)
t_combined = timeit.timeit(lambda: extract_all_combined(sample_text), number=500)

print(f"\n   Multi-pattern extraction (500 iterations):")
print(f"   Separate patterns (4 passes): {t_separate:.3f}s")
print(f"   Combined pattern (1 pass):    {t_combined:.3f}s")
print(f"   Speedup: {t_separate/t_combined:.1f}x")

sep = multi_pattern_match(sample_text, PATTERNS_SEPARATE)
com = extract_all_combined(sample_text)
print(f"\n   Results comparison:")
for key in sep:
    sep_set = set(sep[key])
    com_set = set(com[key])
    match = '==' if sep_set == com_set else '!='
    print(f"   {key}: {len(sep[key])} separate {match} {len(com[key])} combined")
```

---

## 95.4 สรุป Part 95

```
Regex Performance Optimization:

1. Always compile:
   re.compile() once at module level, reuse across calls
   Python caches ~512 patterns automatically, but explicit compile is clearer
   Module-level constants are the best practice

2. Anchoring strategies:
   ^ and $ at start/end: fail fast on non-matching strings
   \A and \Z: absolute start/end (unaffected by MULTILINE flag)
   \b for word boundaries: avoids scanning mid-word

3. Pattern specificity:
   Most specific first: [A-Z]{3} before .{3}
   Character classes > alternation: [aeiou] vs (a|e|i|o|u)
   Avoid .*: use [^delimiter]* or more specific class

4. Non-capturing groups:
   (?:...) instead of (...) when you don't need the captured text
   Reduces match object overhead and memory allocation
   Measurable difference in tight loops over large text

5. Pre-filtering:
   Python check (str method) before regex when possible
   'keyword' in text before re.search(pattern, text)
   isdigit()/isalpha() can rule out 80% of lines cheaply

6. Combined vs separate patterns:
   Single combined alternation pattern = one pass through text
   Multiple separate patterns = N passes through same text
   Tradeoff: combined is faster but harder to maintain

7. Generator/streaming:
   Process large files line by line with generators
   Avoid loading entire file into memory
   yield from pattern match vs list comprehension for large datasets
```

---

*[← Part 94: Regex in DevOps & Infrastructure](part-94-devops.md) | [→ Part 96: Regex for Network Protocol Analysis](part-96-network-protocols.md)*
