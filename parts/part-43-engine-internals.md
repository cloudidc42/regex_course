# Part 43: Regex Engine Internals — NFA, DFA, Backtracking

> **ระดับ:** สูง-ผู้เชี่ยวชาญ | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-42

---

## 43.1 NFA vs. DFA Fundamentals

```python
"""
Regex Engine Types:
===================

1. DFA (Deterministic Finite Automaton)
   - ทุก state มี transition ที่แน่นอน
   - ไม่มี backtracking
   - เวลา O(n) เสมอ (n = ความยาว input)
   - ไม่รองรับ backreferences, lookaround
   - เช่น: RE2, Hyperscan, awk
   
2. NFA (Non-deterministic Finite Automaton)
   - หลาย state พร้อมกันได้
   - มี backtracking (Thompson NFA = lazy simulation)
   - เวลา O(n) average, O(2^n) worst case
   - รองรับ backreferences, lookaround
   - เช่น: Python re, Java, .NET, PHP (PCRE), Perl
   
3. Hybrid (NFA → DFA compilation)
   - compile เป็น DFA ล่วงหน้า
   - เร็วสำหรับ patterns ที่ compile ได้
   - ตกกลับเป็น NFA สำหรับ features พิเศษ
   - เช่น: Go regexp, RE2
"""

import re
import time

# ===== Demonstrate NFA backtracking behavior =====

def count_steps(pattern_str: str, text: str) -> dict:
    """จำลองการนับ backtracking steps"""
    steps = {'count': 0}
    
    class CountingPattern:
        def __init__(self, pattern):
            self.pattern = re.compile(pattern)
            self.call_count = 0
        
        def match(self, text):
            self.call_count += 1
            return self.pattern.match(text)
    
    return {'pattern': pattern_str, 'text_len': len(text)}


print("NFA vs DFA Characteristics:")
print("=" * 60)

# ===== Exponential backtracking demo =====
def measure_time(fn, *args):
    start = time.perf_counter_ns()
    result = fn(*args)
    return (time.perf_counter_ns() - start) / 1000, result

# NFA backtracking: (a+)+ on non-matching input
print("\n1. Exponential Backtracking ((a+)+$):")
print(f"   {'n':>4} {'time(μs)':>12} {'pattern'}")
for n in [5, 10, 15, 20, 25]:
    text = 'a' * n + '!'  # won't match → maximum backtracking
    pattern = re.compile(r'^(a+)+$')
    elapsed, _ = measure_time(pattern.match, text)
    bar = '█' * min(int(elapsed / 10), 30)
    print(f"   {n:>4} {elapsed:>12.1f}μs {bar}")

print("\n   Optimized (no nested quantifier):")
for n in [5, 10, 15, 20, 25]:
    text = 'a' * n + '!'
    pattern = re.compile(r'^a+$')
    elapsed, _ = measure_time(pattern.match, text)
    print(f"   {n:>4} {elapsed:>12.1f}μs")


# ===== Thompson NFA simulation =====
print("\n2. Thompson NFA State Sets:")
print("   Pattern: a(b|c)*d")
print("   Input:   abcbd")
print()
print("   Char  States active (NFA simulation)")
print("   ----  ----------------------------------")

# Manually trace NFA states for a(b|c)*d
trace = [
    ('',  'S0 (start)'),
    ('a', 'S1 (after a)'),
    ('b', 'S2 (after b|c), S3 (loop b|c)'),
    ('c', 'S2 (after b|c), S3 (loop b|c)'),
    ('b', 'S2 (after b|c), S3 (loop b|c)'),
    ('d', 'S4 (ACCEPT)'),
]

for char, states in trace:
    char_display = f"'{char}'" if char else 'start'
    print(f"   {char_display:<6} {states}")
```

---

## 43.2 PCRE vs. RE2 vs. Python re

```python
import re
import sys

print("\nRegex Engine Comparison:")
print("=" * 60)

ENGINE_FEATURES = {
    'Python re': {
        'engine': 'NFA (backtracking)',
        'backreferences': True,
        'lookahead': True,
        'lookbehind': 'fixed-width only',
        'atomic_groups': False,
        'possessive': False,
        'unicode': True,
        'worst_case': 'O(2^n)',
        'notes': 'Standard library, regex module adds extensions',
    },
    'PCRE2': {
        'engine': 'NFA + optimizations',
        'backreferences': True,
        'lookahead': True,
        'lookbehind': 'variable-width',
        'atomic_groups': True,
        'possessive': True,
        'unicode': True,
        'worst_case': 'O(2^n) without atomic',
        'notes': 'PHP, Ruby, most languages',
    },
    'RE2 (Go)': {
        'engine': 'DFA/NFA hybrid',
        'backreferences': False,
        'lookahead': True,
        'lookbehind': 'positive fixed-width',
        'atomic_groups': False,
        'possessive': False,
        'unicode': True,
        'worst_case': 'O(n)',
        'notes': 'Google RE2: guaranteed linear time',
    },
    'Oniguruma': {
        'engine': 'NFA',
        'backreferences': True,
        'lookahead': True,
        'lookbehind': 'variable-width',
        'atomic_groups': True,
        'possessive': False,
        'unicode': True,
        'worst_case': 'O(2^n)',
        'notes': 'Ruby default, mbstring PHP',
    },
}

for engine, features in ENGINE_FEATURES.items():
    print(f"\n  {engine}:")
    for key, val in features.items():
        indicator = '✓' if val is True else '✗' if val is False else '~' if isinstance(val, str) and 'only' in str(val).lower() else '-'
        print(f"    {key:<20}: {indicator} {val}")


# ===== Python re module internals =====
print("\n\nPython re Module Details:")
print("-" * 40)
print(f"  Python version: {sys.version.split()[0]}")
print(f"  re module: backtracking NFA")
print(f"  Pattern cache size: {getattr(re, '_MAXCACHE', 'N/A')}")

# regex module availability
try:
    import regex
    print(f"\n  regex module: AVAILABLE")
    print(f"  regex extras: atomic groups, possessive, variable lookbehind")
    print(f"  Install: pip install regex")
except ImportError:
    print(f"\n  regex module: NOT INSTALLED (pip install regex)")


# ===== Choosing the right engine =====
print("\n\nChoosing Engine:")
CHOICE_GUIDE = [
    ("Need backreferences?",           "Use PCRE/Python re/regex"),
    ("Need variable lookbehind?",       "Use PCRE/regex module"),
    ("Processing untrusted input?",     "Use RE2 or set timeout"),
    ("Performance critical?",           "Use RE2 or pre-filter"),
    ("Need Unicode categories?",        "Use regex module or PCRE2"),
    ("Simple validation?",              "Any engine works"),
    ("Building WAF/security tool?",     "RE2 preferred (no ReDoS)"),
    ("Parsing structured data?",        "Consider dedicated parser instead"),
]

for question, answer in CHOICE_GUIDE:
    print(f"  {question:<40} → {answer}")
```

---

## 43.3 ReDoS (Regular Expression Denial of Service)

```python
import re
import time
import signal
from typing import Optional

print("\n\nReDoS — Regular Expression Denial of Service:")
print("=" * 60)

# ===== Vulnerable patterns =====
VULNERABLE_PATTERNS = {
    "(a+)+":         "Nested quantifier, exponential on 'aaa...!'",
    "(a|aa)+":       "Overlapping alternation",
    "(a*)*":         "Redundant nested quantifier",
    "([a-z]+)*":     "Nested char class quantifier",
    "(\\w+\\s?)+":   "Common in email validation gone wrong",
}

print("\n  Vulnerable patterns:")
for pattern, desc in VULNERABLE_PATTERNS.items():
    print(f"    {pattern:<20} {desc}")

# ===== Safe equivalents =====
SAFE_REWRITES = [
    ("(a+)+$",         "a+$",           "Remove redundant nesting"),
    ("(a|aa)+$",       "[a]+$",         "Combine alternation into class"),
    ("([a-z]+\\s*)+$", "[a-z\\s]+$",    "Flatten nested"),
    ("(\\w+\\s?)+$",   "\\w+(?:\\s\\w+)*$", "Explicit structure"),
]

print("\n  Safe rewrites:")
for bad, good, reason in SAFE_REWRITES:
    print(f"    {bad:<25} -> {good:<25} ({reason})")

# ===== ReDoS detection heuristics =====
def detect_redos_risk(pattern: str) -> list:
    """Heuristic: detect potential ReDoS patterns"""
    warnings = []
    
    # Check for nested quantifiers
    if re.search(r'\([^)]*[+*][^)]*\)[+*?]', pattern):
        warnings.append("Potential nested quantifier: (...)+ or (...)*")
    
    # Check for alternation with common parts
    if re.search(r'\([^)]*\|[^)]*\)[+*]', pattern):
        warnings.append("Alternation with quantifier: (a|b)+")
    
    # Check for overlapping character classes
    if re.search(r'\[[^\]]+\][+*]\s*\[[^\]]+\][+*]', pattern):
        warnings.append("Adjacent quantified classes")
    
    # Catastrophically bad: (\w+)+ style
    if re.search(r'\(\\?w\+\)\+', pattern) or re.search(r'\([^)]*\+[^)]*\)\+', pattern):
        warnings.append("CRITICAL: (\\w+)+ pattern detected!")
    
    return warnings


print("\n  ReDoS Risk Detection:")
test_patterns = [
    r'^(\w+)+$',
    r'^([a-z]+\s?)+$',
    r'^[a-z]+$',
    r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
    r'^(a|b|c)+$',
]

for p in test_patterns:
    risks = detect_redos_risk(p)
    status = "⚠ RISKY" if risks else "✓ OK"
    print(f"\n    {p!r}")
    print(f"    Status: {status}")
    for r in risks:
        print(f"    Warning: {r}")


# ===== Timeout protection =====
class SafeRegex:
    """Regex with timeout protection"""
    
    def __init__(self, pattern: str, timeout_ms: int = 100):
        self._pattern = re.compile(pattern)
        self._timeout_ms = timeout_ms
    
    def match(self, text: str) -> Optional[re.Match]:
        """Match with timeout (Unix only via signal)"""
        try:
            # Thread-based timeout alternative
            import threading
            result = [None]
            exception = [None]
            
            def run():
                try:
                    result[0] = self._pattern.match(text)
                except Exception as e:
                    exception[0] = e
            
            thread = threading.Thread(target=run, daemon=True)
            thread.start()
            thread.join(timeout=self._timeout_ms / 1000)
            
            if thread.is_alive():
                return None  # Timeout
            if exception[0]:
                raise exception[0]
            return result[0]
        except Exception:
            return None
    
    def is_safe(self, text: str) -> bool:
        """ตรวจว่า match ได้ใน timeout"""
        return self.match(text) is not None


print("\n  Safe Regex (with timeout):")
safe = SafeRegex(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', timeout_ms=50)
tests = ['user@example.com', 'invalid', 'a' * 100 + '@']
for t in tests:
    result = safe.match(t)
    print(f"    {t[:40]!r:<45} -> {'match' if result else 'no match / timeout'}")
```

---

## 43.4 สรุป Part 43

```
NFA vs DFA:
NFA: backtracking, supports backrefs/lookaround, O(2^n) worst case
DFA: no backtracking, limited features, O(n) guaranteed

Engines:
Python re  = NFA (backtracking), no atomic/possessive
PCRE2      = NFA + JIT, full features, variable lookbehind
RE2        = DFA/NFA hybrid, linear time, no backrefs
Oniguruma  = NFA, full features, Ruby/PHP

ReDoS causes:
- Nested quantifiers: (a+)+, (\\w+\\s?)+
- Overlapping alternation: (a|aa)+
- No anchor on backtracking pattern

Prevention:
1. Use RE2 for untrusted input processing
2. Add timeout to regex operations
3. Pre-validate input length
4. Rewrite: (\\w+\\s?)+ -> \\w+(?:\\s\\w+)*
5. Test with: 'a' * n + '!' and measure time

Python regex alternatives:
- regex module: atomic groups (?>...), possessive *+
- timeout with threading
- input length limits: if len(text) > MAX: reject
```

---

*[← Part 42: Real-world Projects](part-42-projects.md) | [→ Part 44: Unicode & Internationalization](part-44-unicode.md)*
