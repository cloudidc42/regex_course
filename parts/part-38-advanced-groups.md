# Part 38: Advanced Groups — Groups, Backreferences, Atomic Groups

> **ระดับ:** สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-37

---

## 38.1 Non-capturing and Named Groups

```python
import re

# ===== Non-capturing group (?:...) =====
# ใช้จัดกลุ่มโดยไม่ต้องการ capture

patterns = {
    "with capture":    r'(\d{4})-(\d{2})-(\d{2})',
    "non-capturing":   r'(?:\d{4})-(?:\d{2})-(?:\d{2})',
    "mixed":           r'(\d{4})-(?:\d{2})-(\d{2})',
}

text = "Meeting on 2024-10-15 and 2025-01-20"
print("Non-capturing Groups:")
print("=" * 60)
for name, pattern in patterns.items():
    matches = re.findall(pattern, text)
    print(f"  {name:<20}: {matches}")

# ===== Named Groups (?P<name>...) =====
DATE_PATTERN = re.compile(
    r'(?P<year>\d{4})-(?P<month>0[1-9]|1[0-2])-(?P<day>0[1-9]|[12]\d|3[01])'
)

print("\nNamed Groups:")
for m in DATE_PATTERN.finditer(text):
    d = m.groupdict()
    print(f"  Found: year={d['year']} month={d['month']} day={d['day']}")

# ===== Inline flags (?flags:...) =====
examples = [
    (r'(?i:hello)', 'Hello WORLD hello', 'case insensitive'),
    (r'(?m:^\w+)', 'line one\nline two\nline three', 'multiline in group'),
    (r'(?s:.+)', 'first\nsecond', 'dotall in group'),
]

print("\nInline Flags:")
for pattern, text, desc in examples:
    m = re.findall(pattern, text)
    print(f"  {desc:<25}: {m[:3]}")
```

---

## 38.2 Backreferences

```python
import re

# ===== Backreferences \\1, \\2, ... =====
# อ้างอิงกลับไปยัง group ที่ match ไว้แล้ว

print("\nBackreferences:")
print("=" * 60)

# หาคำซ้ำ
REPEATED_WORD = re.compile(r'\b(\w+)\s+\1\b', re.IGNORECASE)
text = "the the quick brown fox fox jumped over the lazy dog"
print(f"  Repeated words: {REPEATED_WORD.findall(text)}")

# HTML tag matching
HTML_TAG = re.compile(r'<(\w+)[^>]*>(.*?)</\1>', re.DOTALL)
html = '<p>Hello <b>world</b></p> <div><span>nested</span></div>'
print(f"\n  HTML tags:")
for m in HTML_TAG.finditer(html):
    print(f"    <{m.group(1)}> -> {m.group(2)[:30]!r}")

# Quoted strings
QUOTED = re.compile(r'([\'"])(.*?)\1')
text = '''He said "hello" and she replied \'world\' and "mixed\' is wrong'''
print(f"\n  Quoted strings: {QUOTED.findall(text)}")

# ===== Named Backreferences (?P=name) =====
NAMED_BACK = re.compile(r'(?P<tag>\w+)=(?P=tag)')
config = "host=host debug=debug mode=test timeout=timeout"
print(f"\n  Named backrefs (key=key): {NAMED_BACK.findall(config)}")

# ===== Backreferences in substitution =====
print("\nBackreferences in substitution:")

name_swap = re.sub(r'(\w+),\s*(\w+)', r'\2 \1', 'Smith, John; Doe, Jane')
print(f"  Name swap: {name_swap}")

wrapped = re.sub(r'(\w+)', r'<tag>\1</tag>', 'hello world')
print(f"  Wrap:      {wrapped}")

deduped = re.sub(r'\b(\w+)(\s+\1)+\b', r'\1', 'the the quick brown fox fox', flags=re.IGNORECASE)
print(f"  Deduped:   {deduped}")

title = re.sub(r'\b([a-z])', lambda m: m.group(1).upper(), 'hello world from python')
print(f"  Title:     {title}")
```

---

## 38.3 Atomic Groups and Possessive Quantifiers

```python
import re

try:
    import regex
    ATOMIC_AVAILABLE = True
except ImportError:
    ATOMIC_AVAILABLE = False
    print("Note: 'regex' module not installed, skipping atomic group examples")
    print("      Install with: pip install regex")

if ATOMIC_AVAILABLE:
    print("\nAtomic Groups (?>...):")
    print("=" * 60)
    
    text = "12345"
    
    normal  = regex.search(r'(\d+)\d', text)
    atomic  = regex.search(r'(?>\d+)\d', text)
    
    print(f"  Normal  (\\d+)\\d   on '12345': {normal.group() if normal else 'No match'}")
    print(f"  Atomic (?>\\d+)\\d on '12345': {atomic.group() if atomic else 'No match'}")
    
    normal2 = regex.search(r'\d+\d', text)
    possessive = regex.search(r'\d++\d', text)
    
    print(f"\n  Normal  \\d+\\d    on '12345': {normal2.group() if normal2 else 'No match'}")
    print(f"  Possessive \\d++\\d on '12345': {possessive.group() if possessive else 'No match'}")


import time

def benchmark_regex(pattern: str, text: str, label: str) -> float:
    start = time.perf_counter()
    result = re.search(pattern, text)
    elapsed = (time.perf_counter() - start) * 1000
    print(f"  {label:<35} {'match' if result else 'no match':<8} {elapsed:.3f}ms")
    return elapsed

print("\nBacktracking Performance:")
print("=" * 60)

benchmark_regex(r'^(a+)+$', "a" * 15 + "!", "Potentially bad (a+)+")
benchmark_regex(r'^a+$', "a" * 15 + "!", "Optimized  a+")

SAFE_PATTERNS = {
    'Overlapping (a|ab)+':        (r'(a|ab)+',  r'(?:a+b?)+'       ),
    'Nested (\\w+\\s*)+':         (r'(\w+\s*)+', r'\w+(?:\s+\w+)*' ),
    'Redundant (\\d+)+':          (r'(\d+)+',    r'\d+'             ),
}

print("\nSafe vs. Unsafe Patterns:")
for desc, (unsafe, safe) in SAFE_PATTERNS.items():
    print(f"\n  Scenario: {desc}")
    print(f"    Unsafe: {unsafe}")
    print(f"    Safe:   {safe}")
```

---

## 38.4 Conditional Patterns

```python
import re

print("\nConditional Patterns:")
print("=" * 60)

OPTIONAL_BOLD = re.compile(
    r'(<b>)?'              # group 1: optional opening tag
    r'(.+?)'               # group 2: content
    r'(?(1)</b>|)'         # if group 1 matched, require </b>
)

samples = ['<b>bold text</b>', 'plain text', '<b>unclosed bold']
print("  Optional <b> tag:")
for s in samples:
    m = OPTIONAL_BOLD.match(s)
    if m:
        has_tag = bool(m.group(1))
        content = m.group(2)
        print(f"    {s!r:<30} -> bold={has_tag} content={content!r}")

CURRENCY = re.compile(
    r'(?P<baht>฿)?'
    r'(?(baht)'
    r'(?P<amount>[\d,]+(?:\.\d{2})?)'
    r'|'
    r'(?P<amount2>[\d,]+(?:\.\d{2})?)\s*บาท'
    r')'
)

currency_texts = ['฿1,250.00', '1,250.00 บาท', '999']
print("\n  Currency conditional:")
for t in currency_texts:
    m = CURRENCY.search(t)
    if m:
        amount = m.group('amount') or m.group('amount2')
        symbol = '฿' if m.group('baht') else 'บาท'
        print(f"    {t!r:<20} -> {amount} {symbol}")
    else:
        print(f"    {t!r:<20} -> no match")
```

---

## 38.5 สรุป Part 38

```
Group Types Summary:
(...)         capturing group         -> \1, m.group(1)
(?:...)       non-capturing           -> จัดกลุ่มโดยไม่ capture
(?P<name>...) named group             -> \g<name>, m.group('name')
(?P=name)     named backreference     -> match เหมือนที่ capture ไว้
(?=...)       positive lookahead      -> ดูไปข้างหน้าโดยไม่ consume
(?!...)       negative lookahead      -> ไม่ match หน้า
(?<=...)      positive lookbehind     -> ดูไปข้างหลัง (fixed width)
(?<!...)      negative lookbehind     -> ไม่ match หลัง
(?>...)       atomic group            -> ไม่ backtrack (regex module)
(?(n)y|no)    conditional             -> ตรวจ group n ก่อน
(?flags:...)  inline flags            -> (?i:...) (?m:...) (?s:...)

Quantifier Variants (regex module):
*  (greedy)     -> backtrack ได้
*? (lazy)       -> backtrack ได้, ลองน้อยก่อน
*+ (possessive) -> ไม่ backtrack เลย (เร็วกว่า, อาจ fail)

Catastrophic backtracking:
- หลีกเลี่ยง (a|b)+ ที่ overlap กัน
- หลีกเลี่ยง (\\w+\\s*)+ nested quantifiers
- ใช้ possessive *+ หรือ atomic (?>...) แทน
- ใช้ anchor ^ $ เพื่อจำกัด search space
```

---

*[← Part 37: Frameworks](part-37-frameworks.md) | [→ Part 39: Lookahead & Lookbehind](part-39-lookahead-lookbehind.md)*
