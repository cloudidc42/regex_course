# Part 67: Web Scraping & HTML Parsing with Regex

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-66

---

## 67.1 HTML Structure Extraction

```python
import re
from typing import Dict, List, Optional, Tuple

print("Web Scraping & HTML Parsing:")
print("=" * 60)

print("\n1. HTML tag extraction:")

HTML_TAG        = re.compile(r'<(?P<close>/)?(?P<tag>[a-zA-Z][a-zA-Z0-9]*)(?P<attrs>[^>]*)/?>', re.DOTALL)
HTML_ATTR       = re.compile(r'\s+(?P<name>[a-zA-Z_:][a-zA-Z0-9_:.-]*)(?:\s*=\s*(?P<value>"[^"]*"|\x27[^\x27]*\x27|[^\s>]*))?')
HTML_COMMENT    = re.compile(r'<!--.*?-->', re.DOTALL)
HTML_SCRIPT     = re.compile(r'<script[^>]*>.*?</script>', re.DOTALL | re.IGNORECASE)
HTML_STYLE      = re.compile(r'<style[^>]*>.*?</style>', re.DOTALL | re.IGNORECASE)
HTML_ENTITY     = re.compile(r'&(?:#(\d+)|#x([0-9a-fA-F]+)|([a-zA-Z]+));')
HTML_WHITESPACE = re.compile(r'\s+')


ENTITY_MAP = {
    'amp': '&', 'lt': '<', 'gt': '>', 'quot': '"', 'apos': "'",
    'nbsp': '\xa0', 'copy': '\xa9', 'reg': '\xae', 'trade': '™',
    'mdash': '—', 'ndash': '–', 'hellip': '…', 'euro': '€',
}


def decode_entity(m: re.Match) -> str:
    dec, hex_, name = m.group(1), m.group(2), m.group(3)
    if dec:   return chr(int(dec))
    if hex_:  return chr(int(hex_, 16))
    return ENTITY_MAP.get(name, m.group(0))


def parse_attrs(attr_str: str) -> Dict:
    attrs = {}
    for m in HTML_ATTR.finditer(attr_str):
        name  = m.group('name')
        value = m.group('value')
        if value:
            if value.startswith(('"', "'")) and value.endswith(value[0]):
                value = value[1:-1]
        attrs[name] = value
    return attrs


def html_to_text(html: str) -> str:
    text = HTML_COMMENT.sub('', html)
    text = HTML_SCRIPT.sub('', text)
    text = HTML_STYLE.sub('', text)
    text = HTML_TAG.sub('', text)
    text = HTML_ENTITY.sub(decode_entity, text)
    text = HTML_WHITESPACE.sub(' ', text)
    return text.strip()


html_samples = [
    '<p class="intro" id="p1">Hello <strong>world</strong>!</p>',
    '<a href="https://example.com" target="_blank" rel="noopener">Click me</a>',
    '<img src="/img/logo.png" alt="Logo" width="200" height="100" />',
    '<input type="text" name="email" placeholder="Enter email" required>',
]

print(f"\n   Attribute parsing:")
for html in html_samples:
    m = HTML_TAG.search(html)
    if m:
        tag   = m.group('tag')
        attrs = parse_attrs(m.group('attrs'))
        print(f"\n   <{tag}>")
        for k, v in attrs.items():
            print(f"     {k} = {v!r}")

print(f"\n   HTML to text:")
complex_html = '''
<html>
<head><title>Test Page</title></head>
<body>
  <!-- Navigation -->
  <nav><a href="/">Home</a> | <a href="/about">About</a></nav>
  <h1>Hello &amp; Welcome</h1>
  <p>Price: &euro;100 &mdash; great value!</p>
  <script>alert("no js")</script>
</body>
</html>
'''
print(f"   Text: {html_to_text(complex_html)!r}")
```

---

## 67.2 CSS Selector-like Extraction

```python
import re
from typing import List, Dict, Optional

print("\nCSS-like Extraction Patterns:")
print("=" * 60)

def find_by_tag(html: str, tag: str) -> List[str]:
    pattern = re.compile(
        rf'<{re.escape(tag)}(?:\s[^>]*)?>.*?</{re.escape(tag)}>',
        re.DOTALL | re.IGNORECASE
    )
    return pattern.findall(html)


def find_by_id(html: str, id_val: str) -> Optional[str]:
    pattern = re.compile(
        r'<(?P<tag>[a-zA-Z][a-zA-Z0-9]*)'
        rf'[^>]*\bid\s*=\s*["\x27]?{re.escape(id_val)}["\x27]?[^>]*>'
        r'(?P<content>.*?)'
        r'</(?P=tag)>',
        re.DOTALL | re.IGNORECASE
    )
    m = pattern.search(html)
    return m.group('content') if m else None


def find_by_class(html: str, class_name: str) -> List[str]:
    pattern = re.compile(
        r'<(?P<tag>[a-zA-Z][a-zA-Z0-9]*)'
        rf'[^>]*\bclass\s*=\s*["\x27][^"\x27]*\b{re.escape(class_name)}\b[^"\x27]*["\x27][^>]*>'
        r'(?P<content>.*?)'
        r'</(?P=tag)>',
        re.DOTALL | re.IGNORECASE
    )
    return [m.group('content').strip() for m in pattern.finditer(html)]


def find_meta(html: str) -> Dict[str, str]:
    META_RE    = re.compile(r'<meta\s+(?P<attrs>[^>]+)>', re.IGNORECASE)
    NAME_RE    = re.compile(r'\bname\s*=\s*["\x27]([^"\x27]+)["\x27]', re.IGNORECASE)
    PROPERTY_RE = re.compile(r'\bproperty\s*=\s*["\x27]([^"\x27]+)["\x27]', re.IGNORECASE)
    CONTENT_RE  = re.compile(r'\bcontent\s*=\s*["\x27]([^"\x27]*)["\x27]', re.IGNORECASE)
    meta = {}
    for m in META_RE.finditer(html):
        attrs = m.group('attrs')
        key_m = NAME_RE.search(attrs) or PROPERTY_RE.search(attrs)
        val_m = CONTENT_RE.search(attrs)
        if key_m and val_m:
            meta[key_m.group(1)] = val_m.group(1)
    return meta


web_page = '''
<!DOCTYPE html>
<html lang="en">
<head>
  <meta name="description" content="A sample product page">
  <meta name="keywords" content="python, regex, tutorial">
  <meta property="og:title" content="Regex Course">
  <meta property="og:url" content="https://example.com/regex">
  <title>Products</title>
</head>
<body>
  <h1 id="main-title">Best Products</h1>
  <div class="product">
    <h2 class="title">Python Book</h2>
    <span class="price">$29.99</span>
    <p class="desc">Learn Python with regex</p>
  </div>
  <div class="product">
    <h2 class="title">Regex Guide</h2>
    <span class="price">$19.99</span>
    <p class="desc">Master regular expressions</p>
  </div>
  <ul id="features">
    <li>Feature 1</li>
    <li>Feature 2</li>
    <li>Feature 3</li>
  </ul>
</body>
</html>
'''

print(f"\n   By ID 'main-title': {find_by_id(web_page, 'main-title')!r}")
print(f"\n   By class 'title':   {find_by_class(web_page, 'title')}")
print(f"\n   By class 'price':   {find_by_class(web_page, 'price')}")
print(f"\n   Meta tags:")
for k, v in find_meta(web_page).items():
    print(f"   {k:<20} = {v!r}")
```

---

## 67.3 URL and Link Extraction

```python
import re
from typing import List, Dict
from urllib.parse import urljoin

print("\nURL & Link Extraction:")
print("=" * 60)

HREF_RE    = re.compile(r'<a\b[^>]*\bhref\s*=\s*["\x27]([^"\x27]+)["\x27][^>]*>(?P<text>.*?)</a>', re.DOTALL | re.IGNORECASE)
SRC_RE     = re.compile(r'\bsrc\s*=\s*["\x27]([^"\x27]+)["\x27]', re.IGNORECASE)
ACTION_RE  = re.compile(r'<form\b[^>]*\baction\s*=\s*["\x27]([^"\x27]+)["\x27][^>]*>', re.IGNORECASE)

ABS_URL    = re.compile(r'^https?://', re.IGNORECASE)
EMAIL_HREF = re.compile(r'^mailto:', re.IGNORECASE)
TEL_HREF   = re.compile(r'^tel:', re.IGNORECASE)

TAG_STRIP  = re.compile(r'<[^>]+>')


def extract_links(html: str, base_url: str = '') -> Dict:
    links = {'internal': [], 'external': [], 'email': [], 'tel': [], 'assets': []}
    for m in HREF_RE.finditer(html):
        href = m.group(1).strip()
        text = TAG_STRIP.sub('', m.group('text')).strip()
        if EMAIL_HREF.match(href):
            links['email'].append(href[7:])
        elif TEL_HREF.match(href):
            links['tel'].append(href[4:])
        elif ABS_URL.match(href):
            links['external'].append({'url': href, 'text': text})
        else:
            full = urljoin(base_url, href) if base_url else href
            links['internal'].append({'url': full, 'text': text})
    for m in SRC_RE.finditer(html):
        links['assets'].append(m.group(1))
    return links


page_html = '''
<html><body>
  <nav>
    <a href="/">Home</a>
    <a href="/products">Products</a>
    <a href="/about">About Us</a>
    <a href="https://external.com/blog">Blog</a>
    <a href="https://github.com/user/repo">GitHub</a>
  </nav>
  <main>
    <p>Contact: <a href="mailto:info@example.com">Email us</a></p>
    <p>Call: <a href="tel:+66891234567">+66 89 123 4567</a></p>
    <img src="/images/hero.jpg" alt="Hero">
    <script src="/js/app.js"></script>
  </main>
</body></html>
'''

links = extract_links(page_html, 'https://example.com')
print(f"\n   Internal links:")
for L in links['internal']:
    print(f"   {L['url']:<35} '{L['text']}'")
print(f"\n   External links: {[L['url'] for L in links['external']]}")
print(f"   Emails: {links['email']}")
print(f"   Phone:  {links['tel']}")
print(f"   Assets: {links['assets']}")
```

---

## 67.4 Structured Data Extraction

```python
import re
import json
from typing import Dict, List

print("\nStructured Data Extraction:")
print("=" * 60)

JSON_LD  = re.compile(
    r'<script\s+type\s*=\s*["\x27]application/ld\+json["\x27][^>]*>(.*?)</script>',
    re.DOTALL | re.IGNORECASE
)

TABLE_RE  = re.compile(r'<table[^>]*>(.*?)</table>', re.DOTALL | re.IGNORECASE)
TR_RE     = re.compile(r'<tr[^>]*>(.*?)</tr>', re.DOTALL | re.IGNORECASE)
TH_RE     = re.compile(r'<th[^>]*>(.*?)</th>', re.DOTALL | re.IGNORECASE)
TD_RE     = re.compile(r'<td[^>]*>(.*?)</td>', re.DOTALL | re.IGNORECASE)
TAG_CLEAN = re.compile(r'<[^>]+>')

OG_META = re.compile(
    r'<meta\s+property\s*=\s*["\x27]og:([^"\x27]+)["\x27][^>]*content\s*=\s*["\x27]([^"\x27]*)["\x27]',
    re.IGNORECASE
)
OG_META2 = re.compile(
    r'<meta\s+content\s*=\s*["\x27]([^"\x27]*)["\x27][^>]*property\s*=\s*["\x27]og:([^"\x27]+)["\x27]',
    re.IGNORECASE
)


def extract_json_ld(html: str) -> List[Dict]:
    results = []
    for m in JSON_LD.finditer(html):
        try:
            data = json.loads(m.group(1).strip())
            results.append(data)
        except json.JSONDecodeError:
            pass
    return results


def extract_table(html: str) -> List[List[str]]:
    rows = []
    t_m = TABLE_RE.search(html)
    if not t_m:
        return rows
    table_html = t_m.group(1)
    for tr_m in TR_RE.finditer(table_html):
        row_html = tr_m.group(1)
        cells = [TAG_CLEAN.sub('', c.group(1)).strip() for c in TH_RE.finditer(row_html)]
        if not cells:
            cells = [TAG_CLEAN.sub('', c.group(1)).strip() for c in TD_RE.finditer(row_html)]
        if cells:
            rows.append(cells)
    return rows


product_page = '''
<html>
<head>
  <meta property="og:title" content="Python Regex Course">
  <meta property="og:description" content="Learn regex from basics to advanced">
  <meta property="og:image" content="https://example.com/img/course.jpg">
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Course",
    "name": "Python Regex Mastery",
    "description": "Comprehensive regex course",
    "provider": {"@type": "Organization", "name": "Code Academy"},
    "price": "99.00",
    "priceCurrency": "USD"
  }
  </script>
</head>
<body>
  <table>
    <tr><th>Topic</th><th>Duration</th><th>Level</th></tr>
    <tr><td>Basics</td><td>2 hours</td><td>Beginner</td></tr>
    <tr><td>Advanced</td><td>4 hours</td><td>Intermediate</td></tr>
    <tr><td>Security</td><td>3 hours</td><td>Advanced</td></tr>
  </table>
</body>
</html>
'''

ld_data = extract_json_ld(product_page)
if ld_data:
    print(f"\n   JSON-LD data:")
    for key, val in ld_data[0].items():
        if not key.startswith('@'):
            print(f"   {key:<20} = {val!r}")

og_tags = {m.group(1): m.group(2) for m in OG_META.finditer(product_page)}
print(f"\n   Open Graph tags: {og_tags}")

table = extract_table(product_page)
print(f"\n   Table data:")
for row in table:
    print(f"   {row}")
```

---

## 67.5 สรุป Part 67

```
Web Scraping Regex Patterns:

1. HTML tag parsing:
   Tag:   <(/)?(\ w+)([^>]*)>
   Attrs: \s+(\w+)(?:=("[^"]*"|'[^']*'|[^\s>]*))?
   Strip: <[^>]+>

2. Common selectors:
   By ID:    id\s*=\s*["']?VALUE["']?
   By class: class\s*=\s*["'][^"']*\bNAME\b[^"']*["']
   By tag:   <TAG(\s[^>]*)?>.*?</TAG>  (DOTALL)

3. Link types:
   href:    <a[^>]*href\s*=\s*["']([^"']+)["']
   Email:   ^mailto:
   Tel:     ^tel:
   External: ^https?://

4. Structured data:
   JSON-LD: <script type="application/ld+json">(.*?)</script>
   OG tags: <meta property="og:KEY".*?content="VALUE"
   Tables:  <table>...<tr>...<td>...</td>...</tr>...</table>

5. HTML entities:
   Named:   &amp; &lt; &gt; &quot; &nbsp;
   Decimal: &#NNN;
   Hex:     &#xHHH;
   Regex:   &(?:#(\d+)|#x([0-9a-fA-F]+)|([a-zA-Z]+));
```

---

*[← Part 66: Configuration & Formats](part-66-config-formats.md) | [→ Part 68: Log Analysis & Monitoring](part-68-log-analysis.md)*
