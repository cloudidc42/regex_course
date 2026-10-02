# Part 23: Regex Performance — ประสิทธิภาพและการ Optimize

> **ระดับ:** สูง | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-22

---

## 23.1 Regex Engine Types

```
NFA (Non-deterministic Finite Automaton) — Python, Perl, Java, PHP
- Backtracking engine
- ความซับซ้อนที่แย่ที่สุด: O(2^n)
- ยืดหยุ่น: lookaround, backreferences ได้

DFA (Deterministic Finite Automaton) — RE2, awk, grep
- ไม่ backtrack
- O(n) เสมอ, ปลอดภัยกว่า
- ไม่รองรับ backreferences, lookaround
```

---

## 23.2 Catastrophic Backtracking

```python
import re
import time

def timeout_test(pattern_str, test_str):
    p = re.compile(pattern_str)
    start = time.time()
    m = p.match(test_str)
    elapsed = time.time() - start
    return f"Match: {bool(m)}, Time: {elapsed*1000:.1f}ms"

print("Catastrophic Backtracking Demo:")
safe_input = "aaaaaaaab"
print(f"Safe (9 chars): {timeout_test(r'(a+)+b', safe_input)}")

# อันตราย: ยิ่งยาวยิ่งช้า exponential
dangerous = "aaaaaac"  # short demonstration only
print(f"Unsafe 7 chars: {timeout_test(r'(a+)+b', dangerous)}")

# Evil patterns
EVIL_PATTERNS = [
    r'(a+)+',         # nested quantifier
    r'(\w|\w)+',      # overlapping alternation
    r'(a*)*',         # nested star
]
print("\nPotentially Evil Patterns:")
for p in EVIL_PATTERNS:
    print(f"  {p!r}  <- nested/overlapping quantifiers")
```

---

## 23.3 Fixing Catastrophic Backtracking

```python
import re
import time

# Problem: overlapping alternatives (\w|\w)+
p_bad  = re.compile(r'(\w|\w)+')
p_good = re.compile(r'\w+')

text = "hello world"
t1 = time.perf_counter()
for _ in range(10000):
    p_bad.findall(text)
t2 = time.perf_counter()

t3 = time.perf_counter()
for _ in range(10000):
    p_good.findall(text)
t4 = time.perf_counter()

print(f"Overlapping alt: {(t2-t1)*1000:.1f}ms")
print(f"Merged:          {(t4-t3)*1000:.1f}ms")
print(f"Speedup: {(t2-t1)/(t4-t3):.1f}x")

print("\nFix Rules:")
print("  (a+)+  -> a+  (or atomic group)")
print("  (X|X)+ -> X+  (merge overlapping alts)")
print("  (X*)*  -> X*  (flatten)")
print("  (.+)+  -> .+  (flatten)")
```

---

## 23.4 Pre-compile Patterns

```python
import re
import time
from functools import lru_cache

# BAD: re-compiles every call
def find_emails_bad(texts):
    return [re.findall(r'\b[\w.+]+@[\w.]+\.[a-z]{2,}\b', t) for t in texts]

# GOOD: pre-compiled
EMAIL_RE = re.compile(r'\b[\w.+]+@[\w.]+\.[a-z]{2,}\b')
def find_emails_good(texts):
    return [EMAIL_RE.findall(t) for t in texts]

texts = ["user@example.com hello world test"] * 1000

t1 = time.perf_counter()
find_emails_bad(texts[:100])
t2 = time.perf_counter()

t3 = time.perf_counter()
find_emails_good(texts[:100])
t4 = time.perf_counter()

print(f"Without compile: {(t2-t1)*1000:.1f}ms")
print(f"With compile:    {(t4-t3)*1000:.1f}ms")

@lru_cache(maxsize=128)
def get_pattern(template: str, flags: int = 0):
    return re.compile(template, flags)

p1 = get_pattern(r'\b\w+\b')
p2 = get_pattern(r'\b\w+\b')  # from cache
print(f"\nCached same object: {p1 is p2}")
```

---

## 23.5 Alternation Order

```python
import re
import time

# ใส่ frequent alternatives ก่อน
statuses = ['200'] * 100 + ['404'] * 20 + ['301'] * 10 + ['500'] * 5

p_bad  = re.compile(r'500|404|301|200')
p_good = re.compile(r'200|404|301|500')

t1 = time.perf_counter()
for _ in range(10000):
    [p_bad.match(s) for s in statuses]
t2 = time.perf_counter()

t3 = time.perf_counter()
for _ in range(10000):
    [p_good.match(s) for s in statuses]
t4 = time.perf_counter()

print(f"Worst-first: {(t2-t1)*1000:.1f}ms")
print(f"Best-first:  {(t4-t3)*1000:.1f}ms")

# Factor common prefix
p_bad2  = re.compile(r'cat|catch|caught|catalog|category')
p_good2 = re.compile(r'cat(?:ch|alog|egory|(?:aught)?)?')

text = "I have a catalog and category"
print(f"\nBad:  {p_bad2.findall(text)}")
print(f"Good: {p_good2.findall(text)}")
```

---

## 23.6 Greedy vs Lazy vs Specific

```python
import re
import time

text = "<b>bold</b> text <i>italic</i>" * 1000

p_greedy   = re.compile(r'<.*>')
p_lazy     = re.compile(r'<.*?>')
p_specific = re.compile(r'<[^>]+>')

t1 = time.perf_counter()
for _ in range(1000): p_greedy.findall(text)
t2 = time.perf_counter()

t3 = time.perf_counter()
for _ in range(1000): p_lazy.findall(text)
t4 = time.perf_counter()

t5 = time.perf_counter()
for _ in range(1000): p_specific.findall(text)
t6 = time.perf_counter()

print("Greedy vs Lazy vs Specific:")
print(f"  Greedy   <.*>:    {(t2-t1)*1000:.1f}ms")
print(f"  Lazy     <.*?>:   {(t4-t3)*1000:.1f}ms")
print(f"  Specific <[^>]+>: {(t6-t5)*1000:.1f}ms  <- fastest!")

print("\nLazy -> Specific patterns:")
pairs = [
    (r'".*?"',  r'"[^"]*"'),
    (r"'.*?'",  r"'[^']*'"),
    (r'<.*?>',  r'<[^>]+>'),
]
for lazy, specific in pairs:
    print(f"  {lazy} -> {specific}")
```

---

## 23.7 Anchoring

```python
import re
import time

long_text = "x" * 10000 + "target" + "x" * 10000

p_noanchor = re.compile(r'target')
p_anchored = re.compile(r'^target')

t1 = time.perf_counter()
for _ in range(10000): p_noanchor.search(long_text)
t2 = time.perf_counter()

t3 = time.perf_counter()
for _ in range(10000): p_anchored.search(long_text)
t4 = time.perf_counter()

print(f"Without ^: {(t2-t1)*1000:.1f}ms")
print(f"With ^:    {(t4-t3)*1000:.1f}ms")

# re.match() vs re.search() with ^
print("\nAnchoring tips:")
print("  re.match(r'\\w+', text)   is faster than")
print("  re.search(r'^\\w+', text)")
print("\n  \\A = always string start (not affected by MULTILINE)")
print("  \\Z = always string end")
```

---

## 23.8 Performance Best Practices

```python
import re
import timeit

print("Regex Performance Best Practices:")
practices = [
    "1. Pre-compile: re.compile() ล่วงหน้า",
    "2. Avoid nested quantifiers: (a+)+ → a+",
    "3. Negated class: '.*?' → '[^']*'",
    "4. Frequent alternation first: common|rare",
    "5. Anchor when possible: ^, \\A, re.match()",
    "6. str.replace() for literals (faster)",
    "7. RE2 for user-supplied patterns (safe)",
]
for p in practices:
    print(f"  {p}")

# str.replace vs re.sub for literals
text = "Hello World Hello Python" * 10000
t_str = timeit.timeit(lambda: text.replace('Hello', 'Hi'), number=1000)
t_re  = timeit.timeit(lambda: re.sub('Hello', 'Hi', text), number=1000)

print(f"\n  str.replace: {t_str*1000:.1f}ms")
print(f"  re.sub:      {t_re*1000:.1f}ms")
print(f"  str.replace is {t_re/t_str:.1f}x faster for literals")
```

---

## 23.9 สรุป Part 23

```
Performance Rules:
1. Pre-compile patterns ด้วย re.compile()
2. หลีกเลี่ยง (X+)+, (X*)*, (.+)+
3. ใช้ [^X]+ แทน .*? เมื่อรู้ delimiter
4. เรียง alternation: frequent first
5. Anchor ด้วย ^, \A, re.match()
6. str methods สำหรับ literal replacement
7. RE2 สำหรับ user-supplied patterns

Evil Patterns:
(a+)+   (X|X)+   (X*)*   (.+)+
→ O(2^n) เมื่อ fail บน long input
```

---

*[← Part 22: Lookahead & Lookbehind](part-22-lookahead-lookbehind.md) | [→ Part 24: Unicode](part-24-unicode.md)*
