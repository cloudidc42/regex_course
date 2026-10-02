# Part 40: Regex Performance — การเพิ่มประสิทธิภาพ

> **ระดับ:** สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-39

---

## 40.1 Benchmarking and Profiling

```python
import re
import time
import statistics
from typing import List, Callable, Dict, Tuple

def benchmark(fn: Callable, *args, n: int = 1000) -> Dict:
    """Benchmark function: n iterations"""
    times = []
    for _ in range(n):
        start = time.perf_counter_ns()
        fn(*args)
        times.append(time.perf_counter_ns() - start)
    
    return {
        'min_us': min(times) / 1000,
        'avg_us': statistics.mean(times) / 1000,
        'p95_us': sorted(times)[int(n * 0.95)] / 1000,
        'max_us': max(times) / 1000,
    }


def compare_patterns(text: str, patterns: Dict[str, str], n: int = 5000):
    """เปรียบเทียบ performance ของหลาย pattern"""
    compiled = {name: re.compile(p) for name, p in patterns.items()}
    
    print(f"  {'Pattern':<35} {'min':>8} {'avg':>8} {'p95':>8}")
    print(f"  {'-'*35} {'-'*8} {'-'*8} {'-'*8}")
    
    for name, pattern in compiled.items():
        stats = benchmark(pattern.search, text, n=n)
        print(f"  {name:<35} {stats['min_us']:>7.1f}μs {stats['avg_us']:>7.1f}μs {stats['p95_us']:>7.1f}μs")


# ===== Test 1: Compiled vs. Interpreted =====
print("Performance: Compiled vs. Interpreted:")
print("=" * 60)

EMAIL_PATTERN = r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
test_text = "user@example.com and admin@company.org are both valid"

compiled_re = re.compile(EMAIL_PATTERN)

stats_uncompiled = benchmark(lambda: re.search(EMAIL_PATTERN, test_text), n=2000)
stats_compiled   = benchmark(lambda: compiled_re.search(test_text), n=2000)

print(f"\n  Uncompiled re.search: avg={stats_uncompiled['avg_us']:.1f}μs")
print(f"  Compiled   .search:   avg={stats_compiled['avg_us']:.1f}μs")
print(f"  Speedup: {stats_uncompiled['avg_us']/max(stats_compiled['avg_us'],0.001):.1f}x")


# ===== Test 2: Pattern complexity =====
print("\nPattern Complexity:")
compare_patterns(
    text="Hello World 12345 test@example.com 2024-10-15",
    patterns={
        r'simple literal':         r'World',
        r'simple class \w+':       r'\w+',
        r'anchored ^Hello':         r'^Hello',
        r'alternation (a|b|c)':     r'(Hello|World|Python)',
        r'complex email':           r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
        r'nested groups':           r'(\w+)\s+(\w+)\s+(\d+)',
    },
    n=3000
)


# ===== Test 3: Greedy vs. Lazy vs. Character class =====
print("\nGreedy vs. Lazy vs. Char Class:")
html = "<div>Hello <b>World</b> and <i>Python</i></div>" * 10

compare_patterns(
    text=html,
    patterns={
        r'greedy  <.*>':           r'<.*>',
        r'lazy    <.*?>':          r'<.*?>',
        r'char class [^>]+':       r'<[^>]+>',
    },
    n=2000
)
```

---

## 40.2 Optimization Techniques

```python
import re
import time

print("\nOptimization Techniques:")
print("=" * 60)

# ===== 1. Anchoring =====
texts = [
    "This is a short text",
    "A" * 1000 + "needle",
]

print("\n1. Anchoring with ^ reduces backtracking:")
for text in texts:
    preview = text[:30] + '...' if len(text) > 30 else text
    
    without   = benchmark(lambda: re.search(r'needle', text), n=1000)
    with_anch = benchmark(lambda: re.search(r'^needle', text), n=1000)
    
    print(f"   Text: {preview!r}")
    print(f"   Without anchor: {without['avg_us']:.2f}μs")
    print(f"   With anchor:    {with_anch['avg_us']:.2f}μs")
    print()


# ===== 2. Character Classes vs. Alternation =====
print("2. Character Classes vs. Alternation:")
test = "The quick brown fox jumps over the lazy dog 1234567890"

for name, pattern in [
    ('alternation (0|1|2|...)', r'(0|1|2|3|4|5|6|7|8|9)'),
    ('char class [0-9]',         r'[0-9]'),
    ('builtin \\d',              r'\d'),
]:
    stats = benchmark(lambda: re.findall(pattern, test), n=3000)
    print(f"   {name:<35} {stats['avg_us']:.2f}μs")


# ===== 3. Specific vs. General =====
print("\n3. Specific patterns vs. general:")
log_line = '192.168.1.1 - - [02/Oct/2024:10:00:01 +0700] "GET /api/users HTTP/1.1" 200 1234'

GENERAL  = re.compile(r'(.+?) (.+?) (.+?) \[(.+?)\] "(.+?)" (\d+) (\d+)')
SPECIFIC = re.compile(
    r'(\d+\.\d+\.\d+\.\d+) \S+ \S+ \[([^\]]+)\] "([^"]+)" (\d{3}) (\d+)'
)

s_general  = benchmark(lambda: GENERAL.match(log_line), n=3000)
s_specific = benchmark(lambda: SPECIFIC.match(log_line), n=3000)

print(f"   General (.+?):   {s_general['avg_us']:.2f}μs")
print(f"   Specific ([^]+): {s_specific['avg_us']:.2f}μs")
print(f"   Speedup: {s_general['avg_us']/max(s_specific['avg_us'],0.001):.1f}x")


# ===== 4. Pre-filtering =====
print("\n4. Pre-filtering before regex:")
import random

lines = []
for i in range(10000):
    if random.random() < 0.01:
        lines.append(f"2024-10-{i%30+1:02d} ERROR Something went wrong #{i}")
    else:
        lines.append(f"2024-10-{i%30+1:02d} INFO Request processed #{i}")

ERROR_RE = re.compile(r'ERROR (.+) #(\d+)')

start = time.perf_counter_ns()
results_all = [m for line in lines for m in [ERROR_RE.search(line)] if m]
time_all = (time.perf_counter_ns() - start) / 1e6

start = time.perf_counter_ns()
results_filtered = [m for line in lines if 'ERROR' in line for m in [ERROR_RE.search(line)] if m]
time_filtered = (time.perf_counter_ns() - start) / 1e6

print(f"   Without filter: {time_all:.1f}ms  ({len(results_all)} matches)")
print(f"   With 'ERROR' in: {time_filtered:.1f}ms  ({len(results_filtered)} matches)")
print(f"   Speedup: {time_all/max(time_filtered,0.001):.1f}x")
```

---

## 40.3 Optimization Rules

```python
import re

print("\nOptimization Rules:")
rules = [
    ("1. Compile patterns",         "re.compile() at module level, not in loop"),
    ("2. Use anchors",              "^/$ reduce backtracking significantly"),
    ("3. Avoid .* at start",        ".*needle is O(n), \\d+needle is better"),
    ("4. Prefer [^x]+ over .+?",   "Negated class > lazy quantifier"),
    ("5. Pre-filter with 'in'",     "if 'key' in text before regex"),
    ("6. Specific > general",       "\\d{3} is faster than \\d+"),
    ("7. Avoid catastrophic",       "(a|b)+ -> [ab]+ when possible"),
    ("8. Profile before optimizing","measure first, then fix hot paths"),
]

for rule, desc in rules:
    print(f"  {rule:<30} {desc}")
```

---

## 40.4 สรุป Part 40

```
Performance Rules:
1. Compile pattern ครั้งเดียว: re.compile() at module level
2. ใช้ anchor ^ $ เพื่อจำกัด search space
3. [^x]+ เร็วกว่า .+? (negated class > lazy)
4. \d{3} เร็วกว่า \d+ (specific > general)
5. [abc] เร็วกว่า (a|b|c) (class > alternation)
6. pre-filter ด้วย 'text' in string ก่อน regex
7. หลีกเลี่ยง catastrophic backtracking: (a|b)+, (\w+\s*)+

Catastrophic backtracking warning signs:
- Nested quantifiers: (a+)+, (\w+)+
- Overlapping alternation: (a|ab)+
- Patterns ที่ใช้เวลา O(2^n) กับ input ที่ไม่ match

Measurement tools:
- time.perf_counter_ns() สำหรับ micro-benchmarks
- timeit module สำหรับ one-liners
- cProfile + pstats สำหรับ profiling จริง

Python re cache:
- re._MAXCACHE = 512 patterns (Python 3.7+)
- ถ้าใช้มากกว่า 512 pattern แบบ dynamic → เก็บ re.compile() เอง
```

---

*[← Part 39: Lookahead & Lookbehind](part-39-lookahead-lookbehind.md) | [→ Part 41: Testing Regex](part-41-testing.md)*
