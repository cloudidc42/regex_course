# Part 54: Regex Performance Optimization

> **ระดับ:** สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-53

---

## 54.1 Compilation & Caching

```python
import re
import time
from functools import lru_cache

print("Regex Performance — Compilation & Caching:")
print("=" * 60)

print("\n1. Compile once vs compile each call:")

TEXT = "2024-10-15 09:30:00 ERROR Connection failed"
N = 100000

# Bad: recompile every call
start = time.perf_counter_ns()
for _ in range(N):
    m = re.match(r'(\d{4}-\d{2}-\d{2})', TEXT)
bad_ns = (time.perf_counter_ns() - start) / N

# Good: compile once
DATE_RE = re.compile(r'(\d{4}-\d{2}-\d{2})')
start = time.perf_counter_ns()
for _ in range(N):
    m = DATE_RE.match(TEXT)
good_ns = (time.perf_counter_ns() - start) / N

print(f"   re.match(r'pattern', text):  {bad_ns:.1f}ns/call")
print(f"   compiled_re.match(text):     {good_ns:.1f}ns/call")
print(f"   Speedup: {bad_ns/max(good_ns,0.001):.1f}x")

# ===== LRU cache for dynamic patterns =====
print("\n2. LRU cache for dynamic pattern construction:")

@lru_cache(maxsize=128)
def get_word_pattern(word: str) -> re.Pattern:
    return re.compile(r'\b' + re.escape(word) + r'\b', re.IGNORECASE)

words = ["python", "regex", "pattern", "python", "regex"]
text  = "Python is great. I love regex patterns. Python regex!"
for word in words:
    pattern = get_word_pattern(word)
    count = len(pattern.findall(text))
    print(f"   '{word}': {count} occurrences (pattern cached)")

print(f"\n   Cache info: {get_word_pattern.cache_info()}")
```

---

## 54.2 Pattern Optimization

```python
import re
import time

print("\nPattern Optimization:")
print("=" * 60)

print("\n1. Use anchors to reduce search space:")
TEXT = "The quick brown fox jumps over the lazy dog"
N = 100000

r_no_anchor = re.compile(r'The\s+\w+')
r_anchor    = re.compile(r'^The\s+\w+')

start = time.perf_counter_ns()
for _ in range(N):
    r_no_anchor.match(TEXT)
no_anchor_ns = (time.perf_counter_ns() - start) / N

start = time.perf_counter_ns()
for _ in range(N):
    r_anchor.match(TEXT)
anchor_ns = (time.perf_counter_ns() - start) / N

print(f"   Without ^ anchor: {no_anchor_ns:.1f}ns")
print(f"   With ^ anchor:    {anchor_ns:.1f}ns")

print("\n2. Specific patterns are faster:")
r_general  = re.compile(r'<.*?>')
r_specific = re.compile(r'<[^>]*>')

HTML = "<div class='x'><p>Hello</p><span>World</span></div>"
N = 200000

start = time.perf_counter_ns()
for _ in range(N):
    r_general.findall(HTML)
general_ns = (time.perf_counter_ns() - start) / N

start = time.perf_counter_ns()
for _ in range(N):
    r_specific.findall(HTML)
specific_ns = (time.perf_counter_ns() - start) / N

print(f"   <.*?>   (general):  {general_ns:.1f}ns/call")
print(f"   <[^>]*> (specific): {specific_ns:.1f}ns/call")
print(f"   Speedup: {general_ns/max(specific_ns,0.001):.1f}x")

print("\n3. Non-capturing groups (?:...) vs capturing (...):")
r_cap   = re.compile(r'(https?)(://)?([\w.]+)')
r_nocap = re.compile(r'(?:https?)(?://)?(?:[\w.]+)')
URL_TEXT = "Visit https://example.com and http://test.org"
N = 200000

start = time.perf_counter_ns()
for _ in range(N):
    r_cap.findall(URL_TEXT)
cap_ns = (time.perf_counter_ns() - start) / N

start = time.perf_counter_ns()
for _ in range(N):
    r_nocap.findall(URL_TEXT)
nocap_ns = (time.perf_counter_ns() - start) / N

print(f"   Capturing (...)    : {cap_ns:.1f}ns/call")
print(f"   Non-capturing (?:) : {nocap_ns:.1f}ns/call")
print(f"   Speedup: {cap_ns/max(nocap_ns,0.001):.1f}x")
```

---

## 54.3 String vs Regex

```python
import re
import time

print("\nString methods vs Regex:")
print("=" * 60)

print("\nUse str methods when pattern is simple:")
TEXT_SIMPLE = "Hello World Python Regex " * 1000
N = 1000

# str.startswith
r_startswith = re.compile(r'^Hello')
start = time.perf_counter_ns()
for _ in range(N):
    TEXT_SIMPLE.startswith('Hello')
str_start_ns = (time.perf_counter_ns() - start) / N

start = time.perf_counter_ns()
for _ in range(N):
    r_startswith.match(TEXT_SIMPLE)
re_start_ns = (time.perf_counter_ns() - start) / N

print(f"\n   str.startswith():  {str_start_ns:.0f}ns")
print(f"   re.match(r'^Hello'): {re_start_ns:.0f}ns")
print(f"   str method is {re_start_ns/max(str_start_ns,1):.0f}x faster")

# str.count
r_count = re.compile(r'Python')
start = time.perf_counter_ns()
for _ in range(N):
    TEXT_SIMPLE.count('Python')
str_count_ns = (time.perf_counter_ns() - start) / N

start = time.perf_counter_ns()
for _ in range(N):
    len(r_count.findall(TEXT_SIMPLE))
re_count_ns = (time.perf_counter_ns() - start) / N

print(f"\n   str.count('Python'):      {str_count_ns:.0f}ns")
print(f"   len(re.findall(r'Python')): {re_count_ns:.0f}ns")
print(f"   str.count() is {re_count_ns/max(str_count_ns,1):.0f}x faster for exact strings")

print("\nDecision guide:")
decision = [
    ("text.startswith('x')",  "re.match(r'^x', text)",     "starts with"),
    ("text.endswith('.py')",   "re.search(r'\\.py$', text)",  "ends with"),
    ("'substr' in text",       "re.search(r'substr', text)",  "contains"),
    ("text.replace('a','b')",  "re.sub(r'a', 'b', text)",    "simple replace"),
    ("text.split(',')",        "re.split(r',', text)",        "simple split"),
]
print(f"\n   {'Use str':<35} {'Use re':<35} {'When'}")
print(f"   {'-'*35} {'-'*35} {'-'*20}")
for str_way, re_way, when in decision:
    print(f"   {str_way:<35} {re_way:<35} {when}")
```

---

## 54.4 Profiling & Benchmarking

```python
import re
import time

print("\nProfiling Patterns:")
print("=" * 60)

# ===== Email validation approaches =====
print("\n1. Email validation: 3 approaches:")
email = "user.name+tag@example.co.th"
N = 100000

r_simple  = re.compile(r'^[^@]+@[^@]+\.[^@]+$')
r_precise = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
r_strict  = re.compile(
    r'^(?:[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~-]+'
    r'(?:\.[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~-]+)*)'
    r'@(?:(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)*'
    r'[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)$'
)

for label, pat in [("Simple", r_simple), ("Precise", r_precise), ("Strict", r_strict)]:
    start = time.perf_counter_ns()
    for _ in range(N):
        pat.match(email)
    ns = (time.perf_counter_ns() - start) / N
    print(f"   {label:<10}: {ns:.1f}ns/call  match={bool(pat.match(email))}")

print("\n   Recommendation: use 'precise' — good enough + fast")

# ===== findall vs finditer memory =====
print("\n2. findall vs finditer memory comparison:")
BIG_TEXT = "Error at line 42. " * 10000
ERR_RE = re.compile(r'Error at line (\d+)')

start = time.perf_counter_ns()
results = ERR_RE.findall(BIG_TEXT)
findall_ns = (time.perf_counter_ns() - start)

start = time.perf_counter_ns()
count = sum(1 for _ in ERR_RE.finditer(BIG_TEXT))
finditer_ns = (time.perf_counter_ns() - start)

print(f"   Text size: {len(BIG_TEXT):,} chars")
print(f"   findall():  {findall_ns/1e6:.1f}ms (list of {len(results)} items)")
print(f"   finditer(): {finditer_ns/1e6:.1f}ms (iterated {count} items)")
print(f"   For large texts: finditer() avoids building full list in memory")
```

---

## 54.5 สรุป Part 54

```
Performance Rules (in order of impact):

1. Compile patterns once (module level)
   re.compile(r'pattern') → reuse everywhere

2. Use anchors
   ^start_pattern  matches only at beginning
   pattern_end$    matches only at end

3. Use specific character classes
   [^>]*  instead of .*? in HTML  (10x+ faster)
   \d+    instead of .+  for numbers

4. Use non-capturing groups when possible
   (?:...)  instead of (...)  when value not needed

5. Order alternation by frequency
   INFO|WARNING|ERROR|DEBUG  (if INFO is most common)

6. String methods for simple cases
   startswith(), endswith(), in, count() — faster than regex

7. Use finditer() for large text
   Lazy iteration vs. building full list in memory

8. Profile before optimizing
   time.perf_counter_ns() for micro-benchmarks

Pattern complexity vs speed:
Simple literal  'hello'      → fastest (Boyer-Moore)
Char class      [a-z]+       → fast
Anchor          ^hello       → fast
Alternation     a|b|c        → medium
Backreference   (\w+)\s+\1  → slower
Lookaround      (?=...)      → slower
Nested quant    (a+)+        → potential ReDoS
```

---

*[← Part 53: Security Patterns](part-53-security-patterns.md) | [→ Part 55: Regex Testing & Validation](part-55-testing.md)*
