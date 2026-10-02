# Part 63: Advanced Pattern Matching Techniques

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~85 นาที | **ข้อกำหนด:** Part 01-62

---

## 63.1 Lookahead & Lookbehind Mastery

```python
import re
from typing import List, Tuple

print("Advanced Pattern Matching Techniques:")
print("=" * 60)

print("\n1. Complex lookahead combinations:")

PASSWORD_RE = re.compile(
    r'^'
    r'(?=.*[A-Z])'
    r'(?=.*[a-z])'
    r'(?=.*\d)'
    r'(?=.*[!@#$%^&*()_+\-=\[\]{}|;:,.<>?])'
    r'(?!.*\s)'
    r'(?!.*(.)(\1){2,})'
    r'.{10,64}$'
)

passwords = [
    'MyP@ssw0rd!',
    'weakpass',
    'NoSpecial123',
    'Has Space1!',
    'Aaaa1234!@',
    'V3ryStr0ng!@#$',
    'short1!A',
]

print(f"\n   {'Password':<22} {'Valid':<8} {'Reason'}")
print(f"   {'-'*22} {'-'*8} {'-'*30}")
for pwd in passwords:
    m = PASSWORD_RE.match(pwd)
    if m:
        print(f"   {pwd:<22} {'✓ YES':<8}")
    else:
        reasons = []
        if not re.search(r'[A-Z]', pwd): reasons.append('no uppercase')
        if not re.search(r'[a-z]', pwd): reasons.append('no lowercase')
        if not re.search(r'\d', pwd): reasons.append('no digit')
        if not re.search(r'[!@#$%^&*()_+\-=\[\]{}|;:,.<>?]', pwd): reasons.append('no special')
        if re.search(r'\s', pwd): reasons.append('has space')
        if re.search(r'(.)\1{2,}', pwd): reasons.append('repeating chars')
        if len(pwd) < 10: reasons.append('too short')
        print(f"   {pwd:<22} {'✗ NO':<8} {', '.join(reasons)}")


print("\n\n2. Variable-length lookbehind (regex module):")

try:
    import regex

    PRICE_NOT_RANGE = regex.compile(
        r'(?<!(?:from|starting\s+at)\s{1,20})\$\d+(?:\.\d{2})?'
    )
    texts = [
        "The item costs $25.99 and from $10.00",
        "starting at $5.00, main price $89.99",
        "Buy for $49.00 — great value!",
    ]
    print(f"\n   Prices NOT in range context:")
    for text in texts:
        found = PRICE_NOT_RANGE.findall(text)
        print(f"   {text[:55]!r}")
        print(f"   Found: {found}")
except ImportError:
    print("\n   (regex module not available; install with: pip install regex)")
    PRICE_RE = re.compile(r'\$\d+(?:\.\d{2})?')
    RANGE_CTX = re.compile(r'(?:from|starting\s+at)\s+\$\d+')
    text = "costs $25.99 and from $10.00 to $20.00"
    all_prices = set(PRICE_RE.findall(text))
    print(f"\n   All prices (fallback): {all_prices}")
```

---

## 63.2 Atomic Groups & Possessive Quantifiers

```python
import re
import time

print("\nAtomic Groups & Performance:")
print("=" * 60)

print("\n1. Preventing catastrophic backtracking:")

SAFE_ANCHORED = re.compile(r'^a+$')

test_cases = [
    ('aaa', True),
    ('aaab', False),
    ('a' * 15, True),
]

print(f"\n   Safe pattern ^a+$ results:")
for text, expected in test_cases:
    m = SAFE_ANCHORED.match(text)
    result = m is not None
    status = '✓' if result == expected else '✗'
    print(f"   {status} {text!r:<20} → {result}")


try:
    import regex as re2
    print(f"\n   Atomic groups with 'regex' module:")

    ATOMIC = re2.compile(r'^(?>(a+))b$')
    NORMAL = re2.compile(r'^(a+)b$')

    tests = ['aaab', 'aaa', 'ab']
    for t in tests:
        ma = ATOMIC.match(t)
        mn = NORMAL.match(t)
        print(f"   {t!r:<8}: atomic={bool(ma)}, normal={bool(mn)}")

    print(f"\n   Possessive quantifiers (no backtrack):")
    POSS = re2.compile(r'^a++b$')
    NORM = re2.compile(r'^a+b$')
    for t in ['aab', 'aaab', 'aaa']:
        mp = POSS.match(t)
        mn = NORM.match(t)
        print(f"   {t!r:<8}: possessive={bool(mp)}, greedy={bool(mn)}")

except ImportError:
    print("\n   (regex module not available for atomic groups)")
    print("   Standard workarounds:")
    print("   1. Use anchors: ^pattern$ to limit search space")
    print("   2. Avoid (x+)+ nesting — use x+ instead")
    print("   3. Split complex patterns into separate steps")

print("\n\n2. Optimized alternation ordering:")

BAD_ORDER  = re.compile(r'\b(CRITICAL|ERROR|WARNING|DEBUG|INFO)\b')
GOOD_ORDER = re.compile(r'\b(INFO|DEBUG|WARNING|ERROR|CRITICAL)\b')

log_text = ' '.join(['INFO: request processed'] * 200 +
                    ['DEBUG: cache hit'] * 100 +
                    ['WARNING: slow query'] * 50 +
                    ['ERROR: connection failed'] * 20 +
                    ['CRITICAL: disk full'] * 5)

iterations = 5
t0 = time.perf_counter()
for _ in range(iterations):
    BAD_ORDER.findall(log_text)
bad_time = (time.perf_counter() - t0) / iterations * 1000

t0 = time.perf_counter()
for _ in range(iterations):
    GOOD_ORDER.findall(log_text)
good_time = (time.perf_counter() - t0) / iterations * 1000

print(f"\n   Alternation order benchmark ({len(log_text.split())} words):")
print(f"   Rare-first order:   {bad_time:.3f}ms")
print(f"   Common-first order: {good_time:.3f}ms")
improvement = (bad_time - good_time) / bad_time * 100 if bad_time > 0 else 0
print(f"   Improvement: {improvement:.1f}%")
```

---

## 63.3 Conditional Patterns & Backreferences

```python
import re
from typing import Optional, Dict

print("\nConditional Patterns & Backreferences:")
print("=" * 60)

print("\n1. Backreference for matching pairs:")

BALANCED_QUOTES = re.compile(r'(?P<q>["\'])(.+?)(?P=q)')

HTML_TAG_PAIR = re.compile(
    r'<(?P<tag>[a-zA-Z][a-zA-Z0-9]*)(?:\s[^>]*)?>(?P<content>.*?)</(?P=tag)>',
    re.DOTALL
)

texts_with_quotes = [
    '''He said "hello world" and \'goodbye\'.''',
    '''The title is "Python\'s regex" guide.''',
    '''"double" and \'single\' quotes.''',
]
print(f"\n   Balanced quote matching:")
for text in texts_with_quotes:
    matches = BALANCED_QUOTES.findall(text)
    print(f"   {text!r}")
    print(f"   Matches: {[(q, c[:20]) for q, c in matches]}")

print(f"\n   HTML tag pair matching:")
html_samples = [
    '<p>Hello <strong>world</strong>!</p>',
    '<div class="box"><span>content</span></div>',
    '<p>Nested <em>emphasis</em> here</p>',
]
for html in html_samples:
    matches = [(m.group('tag'), m.group('content')[:30]) for m in HTML_TAG_PAIR.finditer(html)]
    print(f"   {html[:55]!r}")
    print(f"   Pairs: {matches}")


print("\n\n2. Conditional pattern (?(name)yes|no):")

PAIRED_BRACKET = re.compile(
    r'(?P<open>[\[({])'
    r'(?P<content>[^\])}]*)'
    r'(?P<close>'
    r'(?(open)'
    r'(?:\]|}|\))'
    r'))'
)

bracket_tests = ['[hello]', '(world)', '{key}', '[unclosed', '(test)']
for t in bracket_tests:
    m = PAIRED_BRACKET.match(t)
    if m:
        print(f"   ✓ {t!r} → open={m.group('open')!r} content={m.group('content')!r} close={m.group('close')!r}")
    else:
        print(f"   ✗ {t!r} → no match")
```

---

## 63.4 Multi-pass Regex Processing

```python
import re
from typing import Dict, List

print("\nMulti-pass Regex Processing:")
print("=" * 60)

print("\n1. Progressive source stripping pipeline:")

BLOCK_COMMENT = re.compile(r'/\*.*?\*/', re.DOTALL)
LINE_COMMENT  = re.compile(r'//[^\n]*')
STRING_LIT    = re.compile(r'"(?:[^"\\]|\\.)*"|\"(?:[^\'\\]|\\.)*\"')
WHITESPACE    = re.compile(r'\s+')


def strip_code_noise(source: str) -> str:
    result = BLOCK_COMMENT.sub(' ', source)
    result = LINE_COMMENT.sub('', result)
    result = re.compile(r'"(?:[^"\\]|\\.)*"').sub('""', result)
    result = WHITESPACE.sub(' ', result).strip()
    return result


def extract_function_calls(clean_source: str) -> List[str]:
    CALL_RE = re.compile(r'([a-zA-Z_$]\w*(?:\.[a-zA-Z_$]\w*)*)\s*\(')
    return sorted(set(CALL_RE.findall(clean_source)))


js_source = '''
/* Config module */
const API_URL = "https://api.example.com"; // base URL

function fetchData(endpoint, options) {
    // Make the request
    const url = `${API_URL}/${endpoint}`;
    return fetch(url, {
        headers: { "Authorization": `Bearer ${TOKEN}` },
        ...options
    }).then(response => response.json());
}
'''

clean = strip_code_noise(js_source)
calls = extract_function_calls(clean)
print(f"\n   Stripped source:\n   {clean[:120]}...")
print(f"\n   Function calls detected: {calls[:8]}")


print("\n\n2. HTML to text conversion pipeline:")

RAW_HTML  = re.compile(r'<[^>]+>')
ENTITIES  = re.compile(r'&(?:#\d+|#x[0-9a-fA-F]+|[a-zA-Z]+);')
EXTRA_WS  = re.compile(r'\s{2,}')

ENTITY_MAP = {
    '&amp;':  '&',  '&lt;': '<',  '&gt;': '>',
    '&quot;': '"', '&apos;': "'", '&nbsp;': ' ',
}

def html_to_text(html: str) -> str:
    text = RAW_HTML.sub('', html)
    def decode_entity(m):
        ent = m.group(0)
        if ent in ENTITY_MAP:
            return ENTITY_MAP[ent]
        code = ent[2:-1]
        try:
            return chr(int(code[1:], 16) if code.startswith('x') else int(code))
        except (ValueError, OverflowError):
            return ent
    text = ENTITIES.sub(decode_entity, text)
    return EXTRA_WS.sub(' ', text).strip()


html_samples = [
    '<h1>Hello &amp; World</h1><p>This is <strong>bold</strong> text.</p>',
    '<a href="https://example.com">Click &lt;here&gt;</a>&nbsp;for more',
    '<div class="container"><p>Price: &#163;100 &#8364;200</p></div>',
]

print(f"\n   HTML to text conversion:")
for html in html_samples:
    text = html_to_text(html)
    print(f"   HTML: {html[:60]!r}")
    print(f"   Text: {text!r}")
    print()
```

---

## 63.5 สรุป Part 63

```
Advanced Pattern Matching:

1. Lookahead combinations:
   (?=.*[A-Z])(?=.*\d)(?!.*\s) → password rules
   Multiple lookaheads AND together

2. Possessive/Atomic (regex module):
   a++ = possessive (no backtrack)
   (?>a+) = atomic group (no backtrack)
   Prevents catastrophic backtracking in
   patterns like (a+)+

3. Alternation optimization:
   Order by frequency: most common first
   INFO|DEBUG|WARNING vs CRITICAL|ERROR|...
   Measurable performance gain on large text

4. Backreferences:
   (?P<q>['"])content(?P=q) → balanced quotes
   <(?P<tag>\w+)>body</(?P=tag)> → HTML pairs
   \\1 or (?P=name) for backward reference

5. Multi-pass processing:
   Pass 1: remove comments
   Pass 2: normalize strings
   Pass 3: extract structure
   Cleaner than one mega-pattern

6. Conditional (?(n)yes|no):
   Available in Python re module
   (?(group_id)pattern_if_matched|pattern_if_not)
```

---

*[← Part 62: Code Analysis](part-62-code-analysis.md) | [→ Part 64: Unicode & Internationalization](part-64-unicode.md)*
