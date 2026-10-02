# Part 49: Atomic Groups & Possessive Quantifiers

> **ระดับ:** สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-48

---

## 49.1 Possessive Quantifiers คืออะไร?

```python
# Possessive quantifiers ป้องกัน backtracking โดยสิ้นเชิง
# Syntax: ?+  *+  ++  {n,m}+  (เพิ่ม + หลัง quantifier)
#
# Standard:   a*   = match as many 'a' as possible, แต่ backtrack ได้
# Possessive: a*+  = match as many 'a' as possible, ไม่ backtrack เลย
#
# ผลลัพธ์: เร็วกว่า + ป้องกัน ReDoS ในบางกรณี
#
# Python re module ไม่รองรับ possessive quantifiers!
# ต้องใช้ regex module (pip install regex)

import sys
print("Possessive Quantifiers (requires regex module):")
print("=" * 60)

try:
    import regex

    print("\n1. Standard vs Possessive quantifier:")

    # Standard: a*b — ถ้า match ไม่ได้ จะ backtrack
    # "aaa" ไม่มี b → a* จับ "aaa" แล้ว fail → backtrack → a* จับ "aa" → fail...
    r_standard   = regex.compile(r'a*b')
    r_possessive = regex.compile(r'a*+b')

    tests = ['aaab', 'aaa', 'b', 'aaabbb', '']
    print(f"\n   {'Text':<12} {'a*b':<10} {'a*+b':<10}")
    print(f"   {'-'*12} {'-'*10} {'-'*10}")
    for t in tests:
        s = r_standard.match(t)
        p = r_possessive.match(t)
        print(f"   {t!r:<12} {str(bool(s)):<10} {str(bool(p)):<10}")

    print("\n   Key difference:")
    print("   - 'aaa' with a*b:  tries a*→'aaa', fail; a*→'aa', fail; ...")
    print("   - 'aaa' with a*+b: tries a*+→'aaa' (possessive, won't give back)")
    print("     b can't match → fail immediately (no backtracking)")

    # ===== Possessive with character class =====
    print("\n2. Possessive with character class:")
    # Match word chars possessively + digits
    r1 = regex.compile(r'\w*+\d')  # possessive \w* then must find digit
    r2 = regex.compile(r'\w*\d')   # standard: backtracks

    tests2 = ['hello123', 'hello', 'abc9', 'nodigits']
    print(f"\n   {'Text':<15} {'\\w*\\d':<10} {'\\w*+\\d':<10}")
    print(f"   {'-'*15} {'-'*10} {'-'*10}")
    for t in tests2:
        s = r2.match(t)
        p = r1.match(t)
        print(f"   {t!r:<15} {str(bool(s)):<10} {str(bool(p)):<10}")

    print("\n   Note: \\w*+\\d usually fails on 'hello123' because")
    print("   \\w*+ consumes ALL word chars including digits,")
    print("   then \\d has nothing left to match")

except ImportError:
    print("\n  regex module not installed. Install with: pip install regex")
    print("  Showing conceptual examples only.\n")
    print("  Possessive syntax:")
    print("    ?+  one or zero, no backtrack")
    print("    *+  zero or more, no backtrack")
    print("    ++  one or more, no backtrack")
    print("    {n,m}+  n to m times, no backtrack")
```

---

## 49.2 Atomic Groups

```python
try:
    import regex

    print("\nAtomic Groups (?>...):")
    print("=" * 60)

    # Atomic group (?>...) = "match this, commit to it, never backtrack into it"
    # Equivalent to possessive for an entire group

    print("\n1. Atomic group basics:")

    # Problem: (?>a|ab)c
    # 'abc' should match (a + bc)? No!
    # (?>a|ab): tries 'a' first (alternation left-to-right), succeeds, commits
    # Then needs 'c' but finds 'b' → fail
    # Atomic: won't try 'ab' alternative → match fails

    r_normal = regex.compile(r'(a|ab)c')    # normal group
    r_atomic = regex.compile(r'(?>a|ab)c')  # atomic group

    tests = ['ac', 'abc']
    print(f"\n   {'Text':<10} {'(a|ab)c':<15} {'(?>a|ab)c':<15}")
    print(f"   {'-'*10} {'-'*15} {'-'*15}")
    for t in tests:
        n = r_normal.match(t)
        a = r_atomic.match(t)
        print(f"   {t!r:<10} {str(bool(n)):<15} {str(bool(a)):<15}")

    print("\n   'abc': normal matches (tries 'a', fails; backtracks to try 'ab')")
    print("   'abc': atomic fails  (tries 'a', commits; no backtrack to 'ab')")

    # ===== Performance benefit =====
    print("\n2. Atomic groups prevent catastrophic backtracking:")

    import time

    # Vulnerable pattern: (a+)+ on string that can't match
    # With atomic: (?>a+)+ — a+ commits, no backtrack within group
    LONG_A = 'a' * 20 + 'b'

    # Simulating with possessive (equivalent to atomic for this case)
    r_vulnerable = regex.compile(r'(a+)+b')
    r_safe       = regex.compile(r'(a++)b')  # possessive inside

    start = time.perf_counter()
    r_safe.match(LONG_A)
    safe_time = time.perf_counter() - start

    start = time.perf_counter()
    r_vulnerable.match(LONG_A)
    vuln_time = time.perf_counter() - start

    print(f"\n   Test string: {'a'*20}b (20 a's + b)")
    print(f"   (a+)+b  (vulnerable): {vuln_time*1000:.2f}ms")
    print(f"   (a++)b  (possessive): {safe_time*1000:.3f}ms")
    print(f"   Speedup: {vuln_time/max(safe_time,0.0001):.0f}x faster")

    # ===== Practical use =====
    print("\n3. Practical: fast identifier matching:")

    # Matching identifiers in code
    IDENT_NORMAL   = regex.compile(r'[a-zA-Z_][a-zA-Z0-9_]*')
    IDENT_ATOMIC   = regex.compile(r'(?>(?:[a-zA-Z_])[a-zA-Z0-9_]*)')
    IDENT_POSSESSIVE = regex.compile(r'[a-zA-Z_][a-zA-Z0-9_]*+')

    code = "myVariable = someFunction(arg1, arg2) + _private_var"
    idents_n = IDENT_NORMAL.findall(code)
    idents_p = IDENT_POSSESSIVE.findall(code)

    print(f"\n   Code: {code!r}")
    print(f"   Identifiers: {idents_n}")
    print(f"   Same result with possessive: {idents_n == idents_p}")

except ImportError:
    print("\n  (regex module not available)")
    print("\n  Atomic group syntax: (?>pattern)")
    print("  - Once matched, never give back characters")
    print("  - Prevents catastrophic backtracking")
    print("  - Useful in alternation: (?>a|ab)")
```

---

## 49.3 Simulating Atomic Groups in Python re

```python
import re

print("\nSimulating Atomic Groups in Python re:")
print("=" * 60)

# Python re ไม่มี atomic groups หรือ possessive quantifiers
# แต่เราสามารถ simulate ได้ด้วย lookahead

# Pattern: atomic(X) ≡ (?=(X))\1
# Trick: lookahead captures X, then backreference matches what lookahead captured
# Since backreference must match exactly what was captured, no backtracking occurs

print("\n1. Atomic group simulation with lookahead:")
print("   (?>X) ≡ (?=(X))\\1")

# Example: (?>a+)b
# Simulate: (?=(a+))\\1b
ATOMIC_SIM = re.compile(r'(?=(a+))\1b')

tests = ['ab', 'aaab', 'aaa', 'b', 'aaabbb']
print(f"\n   {'Text':<12} {'(?>a+)b [sim]'}")
print(f"   {'-'*12} {'-'*15}")
for t in tests:
    m = ATOMIC_SIM.match(t)
    print(f"   {t!r:<12} {bool(m)}")

# ===== Why this works =====
print("\n2. How the simulation works:")
print("""
   (?=(a+))  — lookahead: match a+ and capture in group 1
               The lookahead doesn't advance position
               but stores what a+ matched (possessively in sense of result)
   \\1        — now match exactly what group 1 captured
               If the lookahead captured 'aaa', \\1 must match 'aaa'
               \\1 can't give back chars (it's a fixed string match)
   b         — then must find 'b'
""")

# ===== Practical: IP address without backtracking =====
print("3. Fast IP address matching (atomic simulation):")

# Standard IP
IP_STANDARD = re.compile(
    r'^(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})$'
)

# Atomic-simulated IP (each octet captured atomically)
IP_ATOMIC_SIM = re.compile(
    r'^(?=(\d{1,3}))\1\.(?=(\d{1,3}))\2\.(?=(\d{1,3}))\3\.(?=(\d{1,3}))\4$'
)

ips = ['192.168.1.100', '10.0.0.1', 'invalid.ip', '999.999.999.999', '1.2.3.4']
print(f"\n   {'IP':<25} {'standard':<12} {'atomic sim'}")
print(f"   {'-'*25} {'-'*12} {'-'*12}")
for ip in ips:
    s = IP_STANDARD.match(ip)
    a = IP_ATOMIC_SIM.match(ip)
    print(f"   {ip:<25} {str(bool(s)):<12} {str(bool(a))}")
print("\n   Same results — atomic version prevents unnecessary backtracking")
```

---

## 49.4 When to Use Possessive/Atomic

```python
import re

print("\nWhen to Use Possessive/Atomic Patterns:")
print("=" * 60)

# ===== Safe patterns that benefit from possessive =====
print("\n1. Patterns safe for possessive quantifiers:")
print("""
   Use possessive/atomic when:
   ├── Characters in group can't start what follows
   │   e.g., \\w*+ followed by non-\\w (like @ or space)
   │
   ├── You're matching to end of string/line
   │   e.g., .* at end
   │
   ├── Consecutive non-overlapping tokens
   │   e.g., \\d++ in a lexer
   │
   └── Alternation where first alternative is prefix of second
       e.g., (?>http|https)
""")

# ===== Pattern categories =====
print("2. Safe rewrites:")

SAFE_REWRITES = [
    # (vulnerable, possessive/atomic equivalent, description)
    (r'(\d+)+',      r'(\d++)',      "nested quantifier on digits"),
    (r'(\w+\s?)+',   r'(\w+\s?+)+', "word + optional space"),
    (r'[a-z]+[a-z]+',r'[a-z]++[a-z]+', "consecutive same char class"),
    (r'(.|\n)*',     r'[\s\S]*+',   "match-all across newlines"),
]

print(f"\n   {'Vulnerable':<25} {'Safer':<30} {'Why'}")
print(f"   {'-'*25} {'-'*30} {'-'*30}")
for vuln, safe, desc in SAFE_REWRITES:
    print(f"   {vuln:<25} {safe:<30} {desc}")

# ===== Demonstrating with regex module =====
try:
    import regex
    import time

    print("\n3. Benchmark: standard vs possessive in tight loop:")

    # A more realistic pattern: match a quoted string
    TEXT = '"hello world" + "foo bar" + "baz qux"'

    # Standard quoted string
    r_standard   = regex.compile(r'"[^"]*"')
    # Possessive: [^"]* won't backtrack
    r_possessive = regex.compile(r'"[^"]*+"')

    N = 200000
    start = time.perf_counter_ns()
    for _ in range(N):
        r_standard.findall(TEXT)
    std_ns = (time.perf_counter_ns() - start) / N

    start = time.perf_counter_ns()
    for _ in range(N):
        r_possessive.findall(TEXT)
    pos_ns = (time.perf_counter_ns() - start) / N

    print(f"\n   Text: {TEXT!r}")
    print(f"   \"[^\"]*\"   standard:   {std_ns:.2f}ns/call")
    print(f"   \"[^\"]*+\"  possessive: {pos_ns:.2f}ns/call")
    print(f"   Speedup: {std_ns/max(pos_ns,0.001):.2f}x")
    print("\n   Note: [^\"]*+ is faster because once a non-quote char is")
    print("   consumed it's committed — engine never reconsiders it")

except ImportError:
    pass

# ===== Summary table =====
print("\n4. Feature support by engine:")
SUPPORT_TABLE = [
    ("Python re",    "No",  "No",  "Yes (via lookahead trick)"),
    ("Python regex", "Yes", "Yes", "Yes"),
    ("PCRE/PCRE2",   "Yes", "Yes", "Yes"),
    ("RE2",          "No",  "No",  "No (RE2 uses DFA, no backtracking)"),
    ("JavaScript",   "No",  "No",  "No (ES2018+ only has atomic via workarounds)"),
    ("Java",         "Yes", "No",  "Yes"),
]

print(f"\n   {'Engine':<15} {'Possessive':<12} {'Atomic (?>)':<12} {'Notes'}")
print(f"   {'-'*15} {'-'*12} {'-'*12} {'-'*40}")
for engine, pos, atomic, notes in SUPPORT_TABLE:
    print(f"   {engine:<15} {pos:<12} {atomic:<12} {notes}")
```

---

## 49.5 สรุป Part 49

```
Possessive Quantifiers (regex module only):
?+   {0,1}+   zero or one, no backtrack
*+   {0,}+    zero or more, no backtrack
++   {1,}+    one or more, no backtrack
{n,m}+        n to m times, no backtrack

Atomic Group (regex module only):
(?>pattern)   match pattern, never give back chars

Key properties:
- Once a possessive/atomic group matches, it commits
- The engine NEVER backtracks into it
- Fail fast = much faster on strings that don't match

Python re workaround:
(?>X) ≡ (?=(X))\1  — lookahead captures, backreference matches

When to use:
✓ Pattern can't overlap with what follows
✓ Characters are consumed one-way (e.g., digits in a number)
✓ Performance-critical hot paths
✓ Preventing ReDoS on untrusted input

When NOT to use:
✗ When you genuinely need backtracking for correct matches
✗ When alternatives may consume different amounts (and you need the right one)

Performance gains:
- Possessive on non-overlapping: 10-100x faster on failed matches
- Little difference on successful matches
```

---

*[← Part 48: Scanning & Iterating Matches](part-48-scanning.md) | [→ Part 50: Lookahead & Lookbehind Advanced](part-50-lookahead-advanced.md)*
