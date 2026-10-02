# Part 47: Substitution & Replacement — re.sub() อย่างละเอียด

> **ระดับ:** กลาง | **เวลาเรียน:** ~55 นาที | **ข้อกำหนด:** Part 01-46

---

## 47.1 re.sub() Basics

```python
import re

print("re.sub() — Substitution:")
print("=" * 60)

# ===== Basic substitution =====
print("\n1. Basic substitution:")
text = "Hello World, I love Python and Python is great!"

# Simple replace
result = re.sub(r'Python', 'Regex', text)
print(f"   Original: {text}")
print(f"   Replaced: {result}")

# With count limit
result_n = re.sub(r'Python', 'Regex', text, count=1)
print(f"   First 1:  {result_n}")

# ===== Using groups in replacement =====
print("\n2. Groups in replacement:")

# \1, \2, ... or \g<1>, \g<2>
text = "John Smith, Jane Doe, Bob Jones"
swapped = re.sub(r'(\w+) (\w+)', r'\2, \1', text)
print(f"   Original: {text}")
print(f"   Swapped:  {swapped}")

# Named groups
dates = "Today: 2024-10-15, Tomorrow: 2024-10-16"
reformatted = re.sub(
    r'(?P<y>\d{4})-(?P<m>\d{2})-(?P<d>\d{2})',
    r'\g<d>.\g<m>.\g<y>',
    dates
)
print(f"\n   Dates original:  {dates}")
print(f"   Dates formatted: {reformatted}")

# ===== re.subn() — substitution with count =====
print("\n3. re.subn() — returns (new_string, count):")
text = "The cat sat on the mat near the hat"
result, count = re.subn(r'\b\w+at\b', 'WORD', text)
print(f"   Original: {text}")
print(f"   Result:   {result}")
print(f"   Replaced: {count} occurrences")

# ===== Flags in re.sub() =====
print("\n4. re.sub() with flags:")
text = "PYTHON python Python pYtHoN"
# Case-insensitive replace
result = re.sub(r'python', 'REGEX', text, flags=re.IGNORECASE)
print(f"   {text}")
print(f"   → {result}")
```

---

## 47.2 Callable Replacement Functions

```python
import re
import html
import string

print("\nCallable Replacement (re.sub with function):")
print("=" * 60)

# ===== Function as replacement =====
# re.sub(pattern, fn, string)
# fn receives match object, returns replacement string

print("\n1. Convert case based on match position:")
def title_case_first_word(m):
    """Capitalize first word of each sentence"""
    return m.group().capitalize()

text = "hello world. how are you? fine, thanks!"
result = re.sub(r'(?:^|(?<=[.?!])\s+)\w+', title_case_first_word, text)
print(f"   {text}")
print(f"   → {result}")


print("\n2. Number formatting:")
def format_number(m):
    """Add thousands separators"""
    num = int(m.group())
    return f"{num:,}"

text = "Population: 1234567, Budget: 9876543210"
result = re.sub(r'\d+', format_number, text)
print(f"   {text}")
print(f"   → {result}")


print("\n3. HTML entity encoding:")
def encode_html(m):
    """Encode special HTML characters"""
    return html.escape(m.group())

unsafe_text = "Hello <script>alert('XSS')</script> & \"quotes\""
# Encode only special chars
encoded = re.sub(r'[<>&"\']', encode_html, unsafe_text)
print(f"   Original: {unsafe_text}")
print(f"   Encoded:  {encoded}")


print("\n4. Variable interpolation:")
VARS = {
    'name': 'Alice',
    'lang': 'Python',
    'version': '3.11',
    'os': 'Linux',
}

def interpolate(m):
    key = m.group(1)
    return VARS.get(key, m.group(0))  # keep original if not found

template = "Hello ${name}! You are using ${lang} ${version} on ${os} and ${unknown}."
result = re.sub(r'\$\{(\w+)\}', interpolate, template)
print(f"   Template: {template}")
print(f"   Result:   {result}")


print("\n5. ROT13 cipher with regex:")
def rot13(m):
    char = m.group()
    base = ord('A') if char.isupper() else ord('a')
    return chr((ord(char) - base + 13) % 26 + base)

text = "Hello, World! This is a SECRET message."
encrypted = re.sub(r'[A-Za-z]', rot13, text)
decrypted = re.sub(r'[A-Za-z]', rot13, encrypted)
print(f"   Original:  {text}")
print(f"   Encrypted: {encrypted}")
print(f"   Decrypted: {decrypted}")


print("\n6. Smart unit converter:")
def convert_temp(m):
    value = float(m.group('value'))
    unit = m.group('unit').upper()
    if unit == 'F':
        c = (value - 32) * 5/9
        return f"{c:.1f}°C"
    elif unit == 'C':
        f = value * 9/5 + 32
        return f"{f:.1f}°F"
    return m.group()

TEMP_RE = re.compile(r'(?P<value>-?\d+(?:\.\d+)?)°?(?P<unit>[CF])\b')
text = "Water boils at 100C and freezes at 32F. Body temp is 37C."
result = TEMP_RE.sub(convert_temp, text)
print(f"   {text}")
print(f"   → {result}")
```

---

## 47.3 Advanced Substitution Patterns

```python
import re
from typing import Dict

print("\nAdvanced Substitution Patterns:")
print("=" * 60)

# ===== Code transformation =====
print("\n1. Python 2 → Python 3 migration:")
PY2_PATTERNS = [
    (re.compile(r'\bprint\s+([^(\n]+)'), r'print(\1)'),          # print statement
    (re.compile(r'\braw_input\('), 'input('),                      # raw_input
    (re.compile(r'\.has_key\(([^)]+)\)'), r' in \1'),              # dict.has_key()
    (re.compile(r'\bxrange\('), 'range('),                         # xrange
    (re.compile(r'except\s+(\w+),\s*(\w+):'), r'except \1 as \2:'),  # except syntax
]

py2_code = """
print "Hello World"
for i in xrange(10):
    print i
if d.has_key('key'):
    raw_input("Press enter")
try:
    pass
except ValueError, e:
    pass
"""

py3_code = py2_code
for pattern, replacement in PY2_PATTERNS:
    py3_code = pattern.sub(replacement, py3_code)

print("   Python 2:")
for line in py2_code.strip().splitlines():
    print(f"     {line}")
print("\n   Python 3:")
for line in py3_code.strip().splitlines():
    print(f"     {line}")


# ===== SQL normalization =====
print("\n2. SQL query normalization:")
def normalize_sql(sql: str) -> str:
    # Remove comments
    sql = re.sub(r'--[^\n]*', '', sql)
    sql = re.sub(r'/\*.*?\*/', '', sql, flags=re.DOTALL)
    # Collapse whitespace
    sql = re.sub(r'\s+', ' ', sql).strip()
    # Uppercase keywords
    KEYWORDS = r'\b(SELECT|FROM|WHERE|AND|OR|NOT|IN|JOIN|ON|GROUP BY|ORDER BY|HAVING|LIMIT|INSERT|UPDATE|DELETE|SET|VALUES|CREATE|TABLE|INDEX|DROP|ALTER|ADD|COLUMN)\b'
    sql = re.sub(KEYWORDS, lambda m: m.group().upper(), sql, flags=re.IGNORECASE)
    return sql

queries = [
    "select   id,  name  from  users  where  age > 18",
    """
    select u.name, count(o.id)  -- comment
    from users u  /* join */ join orders o on u.id = o.user_id
    where u.active = 1
    group by u.name
    """,
]

for q in queries:
    normalized = normalize_sql(q)
    print(f"\n   Original: {q.strip()[:60]}...")
    print(f"   Normal:   {normalized[:80]}")


# ===== Markdown to HTML =====
print("\n3. Markdown → HTML converter (basic):")
MARKDOWN_RULES = [
    # Headers
    (re.compile(r'^######\s+(.+)$', re.MULTILINE), r'<h6>\1</h6>'),
    (re.compile(r'^#####\s+(.+)$',  re.MULTILINE), r'<h5>\1</h5>'),
    (re.compile(r'^####\s+(.+)$',   re.MULTILINE), r'<h4>\1</h4>'),
    (re.compile(r'^###\s+(.+)$',    re.MULTILINE), r'<h3>\1</h3>'),
    (re.compile(r'^##\s+(.+)$',     re.MULTILINE), r'<h2>\1</h2>'),
    (re.compile(r'^#\s+(.+)$',      re.MULTILINE), r'<h1>\1</h1>'),
    # Bold and italic
    (re.compile(r'\*\*\*(.+?)\*\*\*'), r'<strong><em>\1</em></strong>'),
    (re.compile(r'\*\*(.+?)\*\*'),     r'<strong>\1</strong>'),
    (re.compile(r'\*(.+?)\*'),         r'<em>\1</em>'),
    (re.compile(r'`(.+?)`'),           r'<code>\1</code>'),
    # Links
    (re.compile(r'\[([^\]]+)\]\(([^)]+)\)'), r'<a href="\2">\1</a>'),
    # Horizontal rule
    (re.compile(r'^---+$', re.MULTILINE), '<hr>'),
]

markdown = """# Hello World

This is **bold** and *italic* text.
Also `inline code` and ***bold italic***.

Check [Python docs](https://python.org) for more.

---

## Section 2

Use `re.sub()` for **text replacement**.
"""

html_result = markdown
for pattern, replacement in MARKDOWN_RULES:
    html_result = pattern.sub(replacement, html_result)

print("   Markdown:")
for line in markdown.strip().splitlines()[:8]:
    print(f"     {line}")
print("\n   HTML:")
for line in html_result.strip().splitlines()[:12]:
    print(f"     {line}")
```

---

## 47.4 Splitting with re.split()

```python
import re

print("\nre.split() Advanced:")
print("=" * 60)

# ===== Basic split =====
print("\n1. Split on multiple delimiters:")
text = "one,two;three|four:five"
parts = re.split(r'[,;|:]', text)
print(f"   {text!r}")
print(f"   → {parts}")

# ===== Split with capturing group (includes separator) =====
print("\n2. Split keeping separator (capturing group):")
text = "word1, word2; word3: word4"
# Use capturing group → separator included in result
parts = re.split(r'([,;:])\s*', text)
print(f"   Parts with sep: {parts}")
# Pair them up
pairs = [(parts[i], parts[i+1]) for i in range(0, len(parts)-1, 2)]
print(f"   Paired: {pairs}")

# ===== Split on whitespace variations =====
print("\n3. Tokenize code:")
code = 'x = func(a, b) + "hello world" + 42'
tokens = re.split(r'(\s+|[+\-*/=(),])', code)
tokens = [t for t in tokens if t and t.strip()]
print(f"   Code: {code!r}")
print(f"   Tokens: {tokens}")

# ===== Split by sentence =====
print("\n4. Sentence tokenization:")
text = """Hello! How are you? I'm fine, thanks. Let's go!
Did you see that? Amazing."""

sentences = re.split(r'(?<=[.!?])\s+', text.replace('\n', ' '))
for i, s in enumerate(sentences, 1):
    print(f"   {i}. {s}")

# ===== re.split vs str.split =====
print("\n5. Performance comparison:")
import time

text = "  foo   bar   baz   qux  " * 1000

# str.split() (no regex)
start = time.perf_counter_ns()
for _ in range(100):
    text.split()
str_time = (time.perf_counter_ns() - start) / 1000

# re.split with \\s+
RE_WS = re.compile(r'\s+')
start = time.perf_counter_ns()
for _ in range(100):
    RE_WS.split(text.strip())
re_time = (time.perf_counter_ns() - start) / 1000

print(f"   str.split():     {str_time:.0f}μs")
print(f"   re.split(\\s+):   {re_time:.0f}μs")
print(f"   str.split() is {re_time/max(str_time,1):.1f}x {'slower' if str_time > re_time else 'faster'}")
print(f"   Rule: str.split() for simple whitespace, re.split() for complex patterns")
```

---

## 47.5 สรุป Part 47

```
re.sub() signatures:
re.sub(pattern, repl, string, count=0, flags=0)
re.subn(pattern, repl, string, count=0, flags=0)  → (result, count)

Replacement types:
1. String literal: r'\1 \g<name>'
2. Function: fn(match) → str

Group references in replacement:
\\1, \\2       numbered group
\\g<1>         same, explicit
\\g<name>      named group
\\g<0>         entire match

Callable replacement:
def replace(m):
    return m.group('name').upper()  # or any transformation

Common patterns:
- Format numbers:       re.sub(r'\\d+', format_fn, text)
- HTML escape:         re.sub(r'[<>&]', escape_fn, text)
- Variable expansion:  re.sub(r'\\$\\{(\\w+)\\}', lookup_fn, text)
- Code migration:      apply multiple (pattern, replacement) pairs

re.split():
- re.split(r'[,;]', text)        → split on char class
- re.split(r'([,;])', text)      → keep separator (capturing group)
- re.split(r'\\s+', text.strip()) → split on any whitespace
- str.split() is faster for simple whitespace
```

---

*[← Part 46: Named Groups & Backreferences](part-46-named-groups.md) | [→ Part 48: Scanning & Iterating Matches](part-48-scanning.md)*
