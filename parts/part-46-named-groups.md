# Part 46: Named Groups & Backreferences — Advanced

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~65 นาที | **ข้อกำหนด:** Part 01-45

---

## 46.1 Named Groups Fundamentals

```python
import re

print("Named Groups — Advanced:")
print("=" * 60)

# ===== Basic named groups =====
# (?P<name>pattern) — define named group
# (?P=name)         — backreference to named group

# IP Address with named parts
IP_PATTERN = re.compile(
    r'^(?P<a>\d{1,3})\.(?P<b>\d{1,3})\.(?P<c>\d{1,3})\.(?P<d>\d{1,3})$'
)

print("\n1. Named groups for IP address:")
ips = ['192.168.1.100', '10.0.0.1', '255.255.255.0', '999.0.0.0']
for ip in ips:
    m = IP_PATTERN.match(ip)
    if m:
        parts = [int(m.group(x)) for x in 'abcd']
        valid = all(0 <= p <= 255 for p in parts)
        print(f"   {ip:<20} octets={parts} valid={valid}")
    else:
        print(f"   {ip:<20} no match")

# ===== groupdict() =====
print("\n2. groupdict() for structured extraction:")
LOG_LINE = re.compile(
    r'(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+'
    r'(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\s+'
    r'\[(?P<module>[^\]]+)\]\s+'
    r'(?P<message>.+)'
)

log_lines = [
    '2024-10-15 09:23:45 ERROR [auth.login] Invalid password for user: alice',
    '2024-10-15 09:23:46 INFO [auth.login] Session created: sess_abc123',
    '2024-10-15 09:25:01 WARNING [db.pool] Connection pool at 90% capacity',
    '2024-10-15 09:30:00 DEBUG [cache.redis] Cache miss for key: user:42',
]

print(f"\n   {'Timestamp':<22} {'Level':<10} {'Module':<15} {'Message'[:30]}")
print(f"   {'-'*22} {'-'*10} {'-'*15} {'-'*30}")
for line in log_lines:
    m = LOG_LINE.match(line)
    if m:
        d = m.groupdict()
        print(f"   {d['timestamp']:<22} {d['level']:<10} {d['module']:<15} {d['message'][:30]}")

# ===== Named groups in replacement =====
print("\n3. Named groups in re.sub():")
# Format: YYYY-MM-DD → DD/MM/YYYY
DATE_REFORMAT = re.compile(r'(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})')
text = "Meeting on 2024-10-15 and 2024-12-31."
reformatted = DATE_REFORMAT.sub(r'\g<day>/\g<month>/\g<year>', text)
print(f"   Original:  {text}")
print(f"   Reformatted: {reformatted}")

# Swap first/last name
NAME_SWAP = re.compile(r'(?P<first>[A-Z][a-z]+)\s+(?P<last>[A-Z][a-z]+)')
names_text = "Alice Smith, Bob Johnson, Carol Williams"
swapped = NAME_SWAP.sub(r'\g<last>, \g<first>', names_text)
print(f"\n   Names original: {names_text}")
print(f"   Names swapped:  {swapped}")
```

---

## 46.2 Backreferences — Forward and Back

```python
import re

print("\nBackreferences:")
print("=" * 60)

# ===== Numbered backreferences =====
print("\n1. Numbered backreferences:")

# Find repeated words
REPEATED_WORD = re.compile(r'\b(\w+)\s+\1\b', re.IGNORECASE)
text = "the the quick brown fox jumps over the the lazy dog"
dupes = REPEATED_WORD.findall(text)
print(f"   Repeated words: {dupes}")

# Remove duplicates
cleaned = REPEATED_WORD.sub(r'\1', text)
print(f"   Cleaned: {cleaned}")

# ===== HTML matching tags =====
print("\n2. Matched HTML tags:")
TAG_MATCHER = re.compile(r'<(\w+)[^>]*>(.*?)</\1>', re.DOTALL)

html_snippets = [
    '<p>Hello World</p>',
    '<div class="test">Content here</div>',
    '<span id="x">Text</span>',
    '<b>Bold <i>italic</i> text</b>',
]

for html in html_snippets:
    m = TAG_MATCHER.match(html)
    if m:
        print(f"   Tag: <{m.group(1)}> Content: {m.group(2)[:30]!r}")

# ===== Named backreferences (?P=name) =====
print("\n3. Named backreferences (?P=name):")

# Quoted strings (same quote type)
QUOTED = re.compile(r'(?P<q>[\'\"]) (?P<content>.*?)(?P=q)')

quoted_texts = [
    'He said "hello" to her',
    "She replied 'goodbye'",
    'Mix "double" and \'single\'',
]

for text in quoted_texts:
    matches = QUOTED.findall(text)
    print(f"   {text!r}")
    print(f"   Quoted: {[content for _, content in matches]}")

# XML-style tag matching
XML_TAG = re.compile(r'<(?P<tag>\w+)[^>]*>(?P<content>.*?)</(?P=tag)>', re.DOTALL)

xml = """
<title>My Title</title>
<h1 class="main">Heading One</h1>
<p id="intro">Intro text.</p>
"""

print(f"\n   XML tag matching:")
for m in XML_TAG.finditer(xml):
    print(f"   <{m.group('tag')}> → {m.group('content')!r}")

# ===== Backreferences in assertions =====
print("\n4. Backreferences + lookahead (duplicate word detection):")
# More sophisticated: find all duplicate word positions
text2 = "The quick quick brown fox fox jumps"
DUPE_FINDER = re.compile(r'\b(\w+)(?=\s+\1\b)', re.IGNORECASE)
positions = [(m.group(1), m.start()) for m in DUPE_FINDER.finditer(text2)]
print(f"   Duplicates found: {positions}")
```

---

## 46.3 Non-Capturing Groups — อย่างละเอียด

```python
import re

print("\nNon-Capturing Groups (?:...):")
print("=" * 60)

# ===== Basic non-capturing =====
# (?:...) groups without capturing → groups() ไม่รวม
print("\n1. Capturing vs Non-capturing:")

text = "2024-10-15"

# With capturing
cap = re.match(r'(\d{4})-(\d{2})-(\d{2})', text)
print(f"   Capturing groups: {cap.groups()}")

# With non-capturing
non_cap = re.match(r'(?:\d{4})-(?:\d{2})-(\d{2})', text)
print(f"   Non-cap + cap: {non_cap.groups()}")  # only day captured

# ===== Alternation grouping =====
print("\n2. Alternation with non-capturing:")

# Protocol part - group for alternation, not capture
URL = re.compile(r'(?:https?|ftp)://(?P<host>[a-zA-Z0-9.-]+)')
for url in ['http://example.com', 'https://api.example.com', 'ftp://files.example.com']:
    m = URL.match(url)
    if m:
        print(f"   {url:<35} host={m.group('host')}")

# ===== Optional groups =====
print("\n3. Optional non-capturing groups:")

# Phone with optional country code
PHONE = re.compile(r'(?:\+?(?P<cc>\d{1,3})\s)?(?P<area>\d{3})[-.](?P<num>\d{3}[-.]\d{4})')
phones = [
    '+1 800-555-1234',
    '800-555-1234',
    '66 89-123-4567',
    '089-123-4567',
]

for ph in phones:
    m = PHONE.match(ph)
    if m:
        d = {k: v for k, v in m.groupdict().items() if v}
        print(f"   {ph:<25} {d}")

# ===== Nested groups =====
print("\n4. Nested groups (capture inside non-capture):")
# Match IPv4 or IPv6 address, capture only the numeric parts
IPv4 = re.compile(
    r'(?:(?P<a>\d{1,3})\.(?P<b>\d{1,3})\.(?P<c>\d{1,3})\.(?P<d>\d{1,3}))'
)
m = IPv4.match('192.168.100.1')
if m:
    print(f"   IPv4 octets: {m.groupdict()}")

# ===== Performance: non-capturing is faster =====
import time

def bench(pattern_str, text, n=100000):
    p = re.compile(pattern_str)
    start = time.perf_counter_ns()
    for _ in range(n):
        p.match(text)
    return (time.perf_counter_ns() - start) / 1000 / n

cap_time   = bench(r'(https?)(://)(\w+\.\w+)', 'https://example.com')
nocap_time = bench(r'(?:https?)(?://)(?:\w+\.\w+)', 'https://example.com')
print(f"\n5. Performance comparison:")
print(f"   Capturing groups:     {cap_time:.2f}μs/call")
print(f"   Non-capturing groups: {nocap_time:.2f}μs/call")
print(f"   Speedup: {cap_time/max(nocap_time,0.001):.1f}x (when groups not needed)")
```

---

## 46.4 Conditional Patterns

```python
import re

print("\nConditional Patterns (?(n)yes|no):")
print("=" * 60)

# ===== Conditional on group number =====
print("\n1. Conditional ?(1) - based on group 1 capture:")
# Match: optional opening bracket, then content, then closing bracket ONLY IF opening existed
BRACKETED = re.compile(r'(\[)?(?P<content>[^\[\]]+)(?(1)\]|)')

tests = [
    '[hello]',
    'hello',
    '[world',    # opening but no closing
]

for t in tests:
    m = BRACKETED.match(t)
    if m:
        print(f"   {t!r:<15} → bracket={bool(m.group(1))}, content={m.group('content')!r}")

# ===== Conditional on named group =====
print("\n2. Conditional on named group:")
# Title or just name
PERSON = re.compile(r'(?:(?P<title>Mr|Mrs|Ms|Dr)\.?\s+)?(?P<name>[A-Z][a-z]+ [A-Z][a-z]+)(?(title)|\s+\(no title\))?')

people = [
    'Dr. Alice Smith',
    'Mr. Bob Johnson',
    'Carol Williams',
]

for p in people:
    m = PERSON.match(p)
    if m:
        d = m.groupdict()
        print(f"   {p!r:<25} title={d['title']!r}, name={d['name']!r}")

# ===== Practical: CSV with optional quotes =====
print("\n3. CSV field with optional quotes:")
CSV_FIELD = re.compile(
    r'(?P<quote>")?'           # optional opening quote
    r'(?(quote)'               # if quoted:
    r'  (?P<qvalue>[^"]*)'     #   match content without quote
    r'|'                       # else (not quoted):
    r'  (?P<value>[^,\n]*)'    #   match content without comma
    r')'
    r'(?(quote)")',            # closing quote only if was opened
    re.VERBOSE
)

csv_tests = [
    '"hello, world"',
    'simple_value',
    '"quoted value"',
    'no quotes here',
]

for csv in csv_tests:
    m = CSV_FIELD.match(csv)
    if m:
        d = m.groupdict()
        value = d['qvalue'] or d['value']
        is_quoted = bool(d['quote'])
        print(f"   {csv!r:<30} quoted={is_quoted}, value={value!r}")


# ===== Using regex module for more conditionals =====
try:
    import regex
    print("\n4. regex module: conditional with named groups:")
    
    # Match date formats: YYYY-MM-DD or DD/MM/YYYY
    DATE_FLEX = regex.compile(r'''
        (?:
            (?P<iso>
                (?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})
            )
            |
            (?P<dmy>
                (?P<day2>\d{2})/(?P<month2>\d{2})/(?P<year2>\d{4})
            )
        )
    ''', regex.VERBOSE)
    
    dates = ['2024-10-15', '15/10/2024', '2024-01-01', '31/12/2024']
    for d in dates:
        m = DATE_FLEX.match(d)
        if m:
            if m.group('iso'):
                print(f"   {d!r} → ISO: year={m.group('year')}, month={m.group('month')}, day={m.group('day')}")
            else:
                print(f"   {d!r} → DMY: day={m.group('day2')}, month={m.group('month2')}, year={m.group('year2')}")

except ImportError:
    print("\n  (regex module not available for conditional groups)")
```

---

## 46.5 สรุป Part 46

```
Named Groups:
(?P<name>...)     define named group
(?P=name)         backreference to named group
\\g<name>         reference in re.sub() replacement
m.group('name')   access by name
m.groupdict()     all named groups as dict

Backreferences:
\\1, \\2, ...      reference to group n
(?P=name)         reference by name (in pattern)
\\g<name>          reference by name (in replacement)

Non-Capturing (?:...):
- Groups without adding to .groups() or .groupdict()
- Used for: alternation, optional parts, repeated patterns
- Slightly faster than capturing (no memory overhead)

Conditional (?(n)yes|no) or (?(name)yes|no):
- Match 'yes' pattern if group n/name was captured
- Match 'no' pattern otherwise
- Useful for: optional brackets, paired delimiters

Performance tips:
- Use (?:...) when you don't need the captured value
- Named groups have same performance as numbered
- Backreferences slightly slow (group lookup at match time)
- Too many groups: consider breaking into multiple patterns
```

---

*[← Part 45: Multiline & DOTALL Flags](part-45-multiline-dotall.md) | [→ Part 47: Substitution & Replacement](part-47-substitution.md)*
