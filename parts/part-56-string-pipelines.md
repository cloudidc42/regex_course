# Part 56: String Processing Pipelines with Regex

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-55

---

## 56.1 Text Normalization Pipeline

```python
import re
from typing import Callable, List

print("Text Normalization Pipeline:")
print("=" * 60)

print("\n1. Build a composable pipeline:")

def make_pipeline(*transforms: Callable[[str], str]) -> Callable[[str], str]:
    def pipeline(text: str) -> str:
        for fn in transforms:
            text = fn(text)
        return text
    return pipeline

def normalize_whitespace(text: str) -> str:
    return re.sub(r'[ \t]+', ' ', text).strip()

def normalize_newlines(text: str) -> str:
    return re.sub(r'\r\n|\r', '\n', text)

def collapse_blank_lines(text: str) -> str:
    return re.sub(r'\n{3,}', '\n\n', text)

def remove_trailing_whitespace(text: str) -> str:
    return re.sub(r'[ \t]+$', '', text, flags=re.MULTILINE)

def normalize_quotes(text: str) -> str:
    text = re.sub(r'[“”„]', '"', text)
    text = re.sub(r"[‘’‚]", "'", text)
    return text

def normalize_dashes(text: str) -> str:
    return re.sub(r'[–—]', '-', text)

def expand_contractions(text: str) -> str:
    contractions = [
        (re.compile(r"\bdon't\b", re.IGNORECASE), "do not"),
        (re.compile(r"\bcan't\b", re.IGNORECASE), "cannot"),
        (re.compile(r"\bwon't\b", re.IGNORECASE), "will not"),
        (re.compile(r"\bit's\b",  re.IGNORECASE), "it is"),
        (re.compile(r"\bI'm\b",   re.IGNORECASE), "I am"),
        (re.compile(r"\bI've\b",  re.IGNORECASE), "I have"),
        (re.compile(r"\bI'll\b",  re.IGNORECASE), "I will"),
        (re.compile(r"\bI'd\b",   re.IGNORECASE), "I would"),
    ]
    for pattern, replacement in contractions:
        text = pattern.sub(replacement, text)
    return text


clean_text = make_pipeline(
    normalize_newlines,
    normalize_whitespace,
    remove_trailing_whitespace,
    collapse_blank_lines,
    normalize_quotes,
    normalize_dashes,
)

sample = '  Hello   World  \n\n\n\nThis   is a test\t\n“Smart quotes” and – dashes — here  '
print(f"\n   Input:  {sample!r}")
print(f"   Output: {clean_text(sample)!r}")

clean_nlp = make_pipeline(clean_text, expand_contractions)
text2 = "I'm sure I don't know what it's about and I can't tell you."
print(f"\n   Input:  {text2!r}")
print(f"   Output: {clean_nlp(text2)!r}")
```

---

## 56.2 Data Extraction Pipeline

```python
import re
from typing import Dict, List
from dataclasses import dataclass, field

print("\nData Extraction Pipeline:")
print("=" * 60)

@dataclass
class ExtractedEntities:
    emails:       List[str] = field(default_factory=list)
    phones:       List[str] = field(default_factory=list)
    urls:         List[str] = field(default_factory=list)
    dates:        List[str] = field(default_factory=list)
    amounts:      List[str] = field(default_factory=list)
    thai_ids:     List[str] = field(default_factory=list)
    ip_addresses: List[str] = field(default_factory=list)

class EntityExtractor:

    EMAIL    = re.compile(r'\b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b')
    PHONE_TH = re.compile(r'\b0[6-9]\d{8}\b')
    URL      = re.compile(r'https?://[^\s<>"\']+')  
    DATE     = re.compile(r'\b\d{4}-\d{2}-\d{2}\b|\b\d{1,2}/\d{1,2}/\d{2,4}\b')
    AMOUNT   = re.compile(r'(?:฿|THB|USD|\$)\s*[\d,]+(?:\.\d{2})?|[\d,]+(?:\.\d{2})?\s*(?:บาท|USD|THB)')
    THAI_ID  = re.compile(r'\b\d{13}\b')
    IP_ADDR  = re.compile(r'\b(?:\d{1,3}\.){3}\d{1,3}\b')

    @classmethod
    def extract(cls, text: str) -> ExtractedEntities:
        result = ExtractedEntities()
        result.emails       = cls.EMAIL.findall(text)
        result.phones       = cls.PHONE_TH.findall(text)
        result.urls         = cls.URL.findall(text)
        result.dates        = cls.DATE.findall(text)
        result.amounts      = cls.AMOUNT.findall(text)
        result.ip_addresses = cls.IP_ADDR.findall(text)
        raw_ids = cls.THAI_ID.findall(text)
        result.thai_ids = [x for x in raw_ids if x not in result.phones]
        return result

    @classmethod
    def redact(cls, text: str) -> str:
        text = cls.EMAIL.sub('[EMAIL]', text)
        text = cls.PHONE_TH.sub('[PHONE]', text)
        text = cls.IP_ADDR.sub('[IP]', text)
        text = cls.THAI_ID.sub('[ID]', text)
        return text


sample_text = """
Customer Report — 2024-10-15

Contact: alice@example.com, phone: 0891234567
Server IP: 192.168.1.100, backup: 10.0.0.5
Order #12345 placed on 15/10/2024
Total: ฿1,250.00 including VAT
Reference ID: 1234567890123

Support URL: https://support.example.com/ticket/999
More info at: http://docs.example.co.th/api
"""

print("\n   Extracted entities:")
entities = EntityExtractor.extract(sample_text)
for field_name in ['emails', 'phones', 'urls', 'dates', 'amounts', 'ip_addresses']:
    values = getattr(entities, field_name)
    if values:
        print(f"   {field_name:<15}: {values}")

print("\n   Redacted text:")
print(EntityExtractor.redact(sample_text))
```

---

## 56.3 Markdown Processing

```python
import re
from typing import List, Tuple

print("\nMarkdown Processing Pipeline:")
print("=" * 60)

print("\n1. Markdown → HTML converter:")

MARKDOWN_RULES: List[Tuple[re.Pattern, str]] = [
    (re.compile(r'^# (.+)$',   re.MULTILINE), r'<h1>\1</h1>'),
    (re.compile(r'^## (.+)$',  re.MULTILINE), r'<h2>\1</h2>'),
    (re.compile(r'^### (.+)$', re.MULTILINE), r'<h3>\1</h3>'),
    (re.compile(r'\*\*\*(.+?)\*\*\*'),   r'<strong><em>\1</em></strong>'),
    (re.compile(r'\*\*(.+?)\*\*'),        r'<strong>\1</strong>'),
    (re.compile(r'\*(.+?)\*'),            r'<em>\1</em>'),
    (re.compile(r'__(.+?)__'),            r'<strong>\1</strong>'),
    (re.compile(r'_(.+?)_'),              r'<em>\1</em>'),
    (re.compile(r'`(.+?)`'),              r'<code>\1</code>'),
    (re.compile(r'!\[([^\]]*)\]\(([^)]+)\)'), r'<img alt="\1" src="\2">'),
    (re.compile(r'\[([^\]]+)\]\(([^)]+)\)'),  r'<a href="\2">\1</a>'),
    (re.compile(r'^---$', re.MULTILINE),  r'<hr>'),
    (re.compile(r'^> (.+)$', re.MULTILINE), r'<blockquote>\1</blockquote>'),
    (re.compile(r'^[*-] (.+)$', re.MULTILINE), r'<li>\1</li>'),
    (re.compile(r'^\d+\. (.+)$', re.MULTILINE), r'<li>\1</li>'),
    (re.compile(r'\n\n'),                 r'</p><p>'),
]

def markdown_to_html(text: str) -> str:
    html = text.strip()
    for pattern, replacement in MARKDOWN_RULES:
        html = pattern.sub(replacement, html)
    return f'<p>{html}</p>'


md_sample = """# Hello World

This is **bold** and *italic* and ***both***.

A `code snippet` inline.

Here's a [link](https://example.com).

> This is a blockquote

- First item
- Second item
"""

html = markdown_to_html(md_sample)
print(f"\n   Markdown → HTML (first 400 chars):")
print(f"   {html[:400]}...")


print("\n\n2. Extract headings from Markdown:")

def extract_headings(text: str) -> List[dict]:
    HEADING = re.compile(r'^(#{1,6})\s+(.+)$', re.MULTILINE)
    headings = []
    for m in HEADING.finditer(text):
        level = len(m.group(1))
        title = m.group(2).strip()
        anchor = re.sub(r'[^\w-]', '-', title.lower())
        anchor = re.sub(r'-+', '-', anchor).strip('-')
        headings.append({'level': level, 'title': title, 'anchor': anchor})
    return headings

headings = extract_headings(md_sample)
print(f"\n   Table of contents:")
for h in headings:
    indent = '  ' * (h['level'] - 1)
    print(f"   {indent}H{h['level']}: {h['title']}")
```

---

## 56.4 Template Engine

```python
import re
from typing import Dict, Any

print("\nSimple Template Engine:")
print("=" * 60)

class TemplateEngine:

    VAR_RE    = re.compile(r'\{\{\s*(\w+)\s*\}\}')
    IF_RE     = re.compile(r'\{%\s*if\s+(\w+)\s*%\}(.*?)\{%\s*endif\s*%\}', re.DOTALL)
    FOR_RE    = re.compile(r'\{%\s*for\s+(\w+)\s+in\s+(\w+)\s*%\}(.*?)\{%\s*endfor\s*%\}', re.DOTALL)
    FILTER_RE = re.compile(r'\{\{\s*(\w+)\s*\|\s*(\w+)\s*\}\}')

    FILTERS = {
        'upper': str.upper,
        'lower': str.lower,
        'title': str.title,
        'strip': str.strip,
        'len':   lambda x: str(len(x)),
    }

    @classmethod
    def render(cls, template: str, context: Dict[str, Any]) -> str:
        def apply_filter(m):
            var, filt = m.group(1), m.group(2)
            value = str(context.get(var, ''))
            fn = cls.FILTERS.get(filt, lambda x: x)
            return fn(value)
        result = cls.FILTER_RE.sub(apply_filter, template)

        def render_for(m):
            item_name, list_name, body = m.group(1), m.group(2), m.group(3)
            items = context.get(list_name, [])
            parts = []
            for item in items:
                inner_ctx = {**context, item_name: item}
                parts.append(cls.render(body, inner_ctx))
            return ''.join(parts)
        result = cls.FOR_RE.sub(render_for, result)

        def render_if(m):
            condition, body = m.group(1), m.group(2)
            return body if context.get(condition) else ''
        result = cls.IF_RE.sub(render_if, result)

        result = cls.VAR_RE.sub(
            lambda m: str(context.get(m.group(1), f'[{m.group(1)}]')),
            result
        )
        return result


template = """Hello, {{ name | title }}!

{% if is_member %}Welcome back, member #{{ member_id }}.
Your email: {{ email | lower }}{% endif %}

Your cart ({{ cart | len }} items):
{% for item in cart %}- {{ item }}
{% endfor %}
Total: {{ total }}"""

context = {
    'name': 'alice smith',
    'is_member': True,
    'member_id': '12345',
    'email': 'Alice@Example.COM',
    'cart': ['Python book', 'Regex guide', 'Coffee'],
    'total': '฿1,250.00',
}

print("\n   Template output:")
print(TemplateEngine.render(template, context))

context2 = {**context, 'is_member': False, 'name': 'guest'}
print("\n   As guest (no member block):")
print(TemplateEngine.render(template, context2))
```

---

## 56.5 สรุป Part 56

```
String Processing with Regex:

1. Pipeline pattern:
   make_pipeline(fn1, fn2, fn3) → composed function
   Each transform: str → str

2. Text normalization:
   Whitespace:    r'[ \t]+'  → ' '
   Newlines:      r'\r\n|\r' → '\n'
   Blank lines:   r'\n{3,}'  → '\n\n'
   Smart quotes:  r'[“”„]' → '"'
   Dashes:        r'[–—]' → '-'

3. Entity extraction:
   Email:    r'\b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b'
   Phone TH: r'\b0[6-9]\d{8}\b'
   URL:      r'https?://[^\s<>"\']+'
   IP:       r'\b(?:\d{1,3}\.){3}\d{1,3}\b'

4. Markdown conversion:
   Apply rules in order (most specific first)
   Headers before bold (# vs ***)
   Images before links (![] vs [])

5. Template engine:
   Variables:    {{ name }}
   Filters:      {{ name | upper }}
   Conditionals: {% if flag %}...{% endif %}
   Loops:        {% for x in items %}...{% endfor %}
   Process: for → if → variables (inner-to-outer)
```

---

*[← Part 55: Regex Testing](part-55-testing.md) | [→ Part 57: Data Pipelines & ETL](part-57-data-pipelines.md)*
