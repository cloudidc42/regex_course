# Part 17: HTML Parsing — การดึงข้อมูลจาก HTML

> **ระดับ:** กลาง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-16

---

## 17.1 ทำไม Regex ไม่เหมาะกับ HTML ทั้งหมด

```
HTML ไม่ใช่ Regular Language — มัน nested และ recursive:
<div>
  <div>
    <div>...</div>   ← nested ลึกแค่ไหนก็ได้
  </div>
</div>

❌ Regex ไม่สามารถ match nested tags ได้อย่างสมบูรณ์
✅ Regex ใช้ได้ดีสำหรับ:
   - ดึง attributes เฉพาะ
   - ดึง URLs จาก href/src
   - Strip HTML tags
   - ค้นหา patterns ใน content
   - Sanitize input
```

---

## 17.2 Strip HTML Tags

```python
import re

def strip_html_tags(html: str, keep_whitespace: bool = False) -> str:
    """Remove all HTML tags"""
    # Remove script and style blocks first
    text = re.sub(r'<(script|style)[^>]*>.*?</\1>', '', html, flags=re.DOTALL | re.IGNORECASE)
    # Remove HTML comments
    text = re.sub(r'<!--.*?-->', '', text, flags=re.DOTALL)
    # Remove remaining tags
    text = re.sub(r'<[^>]+>', '', text)
    # Decode common HTML entities
    entities = {
        '&amp;': '&', '&lt;': '<', '&gt;': '>',
        '&quot;': '"', '&#39;': "'", '&nbsp;': ' ',
        '&copy;': '©', '&reg;': '®', '&trade;': '™',
    }
    for entity, char in entities.items():
        text = text.replace(entity, char)
    text = re.sub(r'&#(\d+);', lambda m: chr(int(m.group(1))), text)
    text = re.sub(r'&#x([0-9a-fA-F]+);', lambda m: chr(int(m.group(1), 16)), text)
    if not keep_whitespace:
        text = re.sub(r'\s+', ' ', text).strip()
    return text


# ทดสอบ
html_samples = [
    "<h1>Hello <b>World</b></h1>",
    "<p>Price: &lt;b&gt;$100&lt;/b&gt; &amp; taxes</p>",
    "<script>alert('xss')</script><p>Safe content</p>",
    """<div class=\"card\">
         <h2>Title</h2>
         <!-- comment -->
         <p>Body text &copy; 2026</p>
       </div>""",
]

print("Strip HTML Tags:")
print("=" * 60)
for html in html_samples:
    clean = strip_html_tags(html)
    print(f"  HTML:  {html[:50]!r}...")
    print(f"  Clean: {clean!r}")
    print()
```

---

## 17.3 Extract URLs from HTML

```python
import re
from urllib.parse import urljoin, urlparse
from typing import List, Dict

class HTMLLinkExtractor:
    """Extract links from HTML"""
    
    # href and src attributes
    HREF_PATTERN = re.compile(
        r'<a\b[^>]*\bhref\s*=\s*["\']([^"\']+)["\'][^>]*>',
        re.IGNORECASE
    )
    SRC_PATTERN = re.compile(
        r'<(?:img|script|iframe|source|track)\b[^>]*\bsrc\s*=\s*["\']([^"\']+)["\'][^>]*>',
        re.IGNORECASE
    )
    ACTION_PATTERN = re.compile(
        r'<form\b[^>]*\baction\s*=\s*["\']([^"\']+)["\'][^>]*>',
        re.IGNORECASE
    )
    
    # All attribute=value URLs
    ANY_URL_ATTR = re.compile(
        r'\b(?:href|src|action|data-url|data-src)\s*=\s*["\']([^"\']+)["\']',
        re.IGNORECASE
    )
    
    @classmethod
    def extract_links(cls, html: str, base_url: str = '') -> List[Dict]:
        links = []
        
        for m in cls.HREF_PATTERN.finditer(html):
            url = m.group(1).strip()
            if url and not url.startswith('#') and not url.startswith('javascript:'):
                absolute = urljoin(base_url, url) if base_url else url
                links.append({'url': absolute, 'type': 'href', 'raw': url})
        
        return links
    
    @classmethod
    def extract_images(cls, html: str, base_url: str = '') -> List[Dict]:
        IMG_PATTERN = re.compile(
            r'<img\b([^>]*)>',
            re.IGNORECASE
        )
        ATTR = re.compile(r'\b(\w[\w-]*)\s*=\s*["\']([^"\']*)["\']')
        
        images = []
        for img_m in IMG_PATTERN.finditer(html):
            attrs = {m.group(1).lower(): m.group(2) for m in ATTR.finditer(img_m.group(1))}
            src = attrs.get('src', '')
            if src:
                images.append({
                    'src': urljoin(base_url, src) if base_url else src,
                    'alt': attrs.get('alt', ''),
                    'width': attrs.get('width', ''),
                    'height': attrs.get('height', ''),
                })
        return images
    
    @classmethod
    def extract_all_urls(cls, html: str) -> List[str]:
        """Extract all URLs from any attribute"""
        return list(set(
            m.group(1) for m in cls.ANY_URL_ATTR.finditer(html)
            if m.group(1) and not m.group(1).startswith('#')
        ))


# ทดสอบ
html_page = """
<html>
<head>
    <link rel="stylesheet" href="/css/style.css">
    <script src="/js/app.js"></script>
</head>
<body>
    <a href="https://example.com">External</a>
    <a href="/about">About</a>
    <a href="#section">Anchor</a>
    <a href="javascript:void(0)">JS link</a>
    <img src="/images/logo.png" alt="Logo" width="200" height="100">
    <img src="https://cdn.example.com/photo.jpg" alt="Photo">
    <form action="/submit" method="post">
        <input type="submit" value="Submit">
    </form>
</body>
</html>
"""

print("HTML Link Extraction:")
print("=" * 60)
print("\nLinks:")
for link in HTMLLinkExtractor.extract_links(html_page, 'https://mysite.com'):
    print(f"  {link['url']}")

print("\nImages:")
for img in HTMLLinkExtractor.extract_images(html_page, 'https://mysite.com'):
    print(f"  src: {img['src']}")
    print(f"   alt: {img['alt']}  {img['width']}x{img['height']}")
```

---

## 17.4 Extract HTML Attributes

```python
import re
from typing import Optional, Dict

class AttributeExtractor:
    """Extract specific attributes from HTML tags"""
    
    @classmethod
    def get_attribute(cls, tag_html: str, attr_name: str) -> Optional[str]:
        """Extract attribute value from tag string"""
        pattern = re.compile(
            rf'\b{re.escape(attr_name)}\s*=\s*(?:"([^"]*)"|\'([^\']*)\'|(\S+))',
            re.IGNORECASE
        )
        m = pattern.search(tag_html)
        if m:
            return m.group(1) or m.group(2) or m.group(3)
        # Boolean attribute (no value)
        bool_pattern = re.compile(rf'\b{re.escape(attr_name)}\b', re.IGNORECASE)
        if bool_pattern.search(tag_html):
            return ''
        return None
    
    @classmethod
    def get_all_attributes(cls, tag_html: str) -> Dict:
        """Extract all attributes from a tag"""
        pattern = re.compile(
            r'\b([\w-]+)\s*=\s*(?:"([^"]*)"|\'([^\']*)\'|(\S+?)(?:\s|>|$))',
            re.IGNORECASE
        )
        attrs = {}
        for m in pattern.finditer(tag_html):
            name = m.group(1).lower()
            value = m.group(2) or m.group(3) or m.group(4) or ''
            attrs[name] = value
        return attrs
    
    @classmethod
    def extract_by_tag_and_attr(cls, html: str, tag: str, attr: str) -> list:
        """Extract all values of attr from specific tag"""
        tag_pattern = re.compile(
            rf'<{re.escape(tag)}\b([^>]*)>',
            re.IGNORECASE
        )
        results = []
        for m in tag_pattern.finditer(html):
            value = cls.get_attribute(m.group(0), attr)
            if value is not None:
                results.append(value)
        return results


# ทดสอบ
test_tags = [
    '<a href="https://example.com" class="link" target="_blank">Click</a>',
    "<img src='/img/photo.jpg' alt='Photo' width=200 height=100 loading=lazy>",
    '<input type="text" name="email" required placeholder="Enter email">',
    '<div data-id="123" data-name="test" class="card active">',
]

print("\nAttribute Extraction:")
print("=" * 60)
for tag in test_tags:
    attrs = AttributeExtractor.get_all_attributes(tag)
    print(f"\n  Tag: {tag[:60]}")
    for name, value in list(attrs.items())[:4]:
        print(f"    {name}: {value!r}")

html = """
<div class="container main">
  <p class="text-lg bold">Hello</p>
  <span class="badge error">Error</span>
  <button class="btn btn-primary">Click</button>
</div>
"""
div_classes = AttributeExtractor.extract_by_tag_and_attr(html, 'div', 'class')
print(f"\nDiv classes: {div_classes}")
```

---

## 17.5 Extract Meta Tags and Head Content

```python
import re
from typing import Dict, Optional

class MetaExtractor:
    """Extract meta tags and head information"""
    
    TITLE_PATTERN = re.compile(r'<title[^>]*>(.*?)</title>', re.IGNORECASE | re.DOTALL)
    META_PATTERN = re.compile(r'<meta\b([^>]*)/?>', re.IGNORECASE)
    
    @classmethod
    def extract(cls, html: str) -> Dict:
        result = {
            'title': '',
            'description': '',
            'keywords': '',
            'author': '',
            'og': {},
            'twitter': {},
            'charset': '',
            'viewport': '',
        }
        
        # Title
        m = cls.TITLE_PATTERN.search(html)
        if m:
            result['title'] = re.sub(r'\s+', ' ', m.group(1)).strip()
        
        # Meta tags
        for m in cls.META_PATTERN.finditer(html):
            tag_content = m.group(1)
            
            name = cls._get_attr(tag_content, 'name')
            prop = cls._get_attr(tag_content, 'property')
            content = cls._get_attr(tag_content, 'content') or ''
            charset = cls._get_attr(tag_content, 'charset')
            
            if charset:
                result['charset'] = charset
            
            if name:
                name_lower = name.lower()
                if name_lower == 'description':
                    result['description'] = content
                elif name_lower == 'keywords':
                    result['keywords'] = content
                elif name_lower == 'author':
                    result['author'] = content
                elif name_lower == 'viewport':
                    result['viewport'] = content
                elif name_lower.startswith('twitter:'):
                    key = name_lower[8:]
                    result['twitter'][key] = content
            
            if prop and prop.startswith('og:'):
                key = prop[3:]
                result['og'][key] = content
        
        return result
    
    @staticmethod
    def _get_attr(tag: str, name: str) -> Optional[str]:
        pattern = re.compile(
            rf'\b{re.escape(name)}\s*=\s*(?:"([^"]*)"|\'([^\']*)\')',
            re.IGNORECASE
        )
        m = pattern.search(tag)
        if m:
            return m.group(1) or m.group(2)
        return None


# ทดสอบ
html_head = """
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>หน้าหลัก | ตัวอย่างเว็บไซต์</title>
    <meta name="description" content="เว็บไซต์ตัวอย่างสำหรับเรียน Regex">
    <meta name="keywords" content="regex, python, tutorial">
    <meta name="author" content="Admin">
    <meta property="og:title" content="Regex Course">
    <meta property="og:description" content="เรียน Regex จากศูนย์ถึงมืออาชีพ">
    <meta property="og:image" content="https://example.com/og.jpg">
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="Regex Course">
</head>
<body></body>
</html>
"""

meta = MetaExtractor.extract(html_head)
print("\nMeta Extraction:")
print(f"  Title:       {meta['title']}")
print(f"  Description: {meta['description']}")
print(f"  Keywords:    {meta['keywords']}")
print(f"  Author:      {meta['author']}")
print(f"  Charset:     {meta['charset']}")
print(f"  OG tags:     {meta['og']}")
print(f"  Twitter:     {meta['twitter']}")
```

---

## 17.6 HTML Sanitizer (XSS Prevention)

```python
import re
from typing import Optional, Dict

class HTMLSanitizer:
    """
    Sanitize HTML to prevent XSS attacks
    For production use, prefer dedicated libraries like bleach
    """
    
    ALLOWED_TAGS = {
        'p', 'br', 'b', 'i', 'em', 'strong', 'u', 's', 'strike',
        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
        'ul', 'ol', 'li', 'dl', 'dt', 'dd',
        'a', 'img', 'blockquote', 'pre', 'code', 'kbd',
        'table', 'thead', 'tbody', 'tr', 'th', 'td',
        'div', 'span', 'hr',
    }
    
    DANGEROUS_PATTERNS = [
        re.compile(r'javascript\s*:', re.IGNORECASE),
        re.compile(r'vbscript\s*:', re.IGNORECASE),
        re.compile(r'on\w+\s*=', re.IGNORECASE),
        re.compile(r'<\s*script', re.IGNORECASE),
        re.compile(r'<\s*/\s*script', re.IGNORECASE),
        re.compile(r'expression\s*\(', re.IGNORECASE),
    ]
    
    @classmethod
    def is_safe_url(cls, url: str) -> bool:
        url = url.strip().lower()
        if url.startswith('javascript:'):
            return False
        if url.startswith('vbscript:'):
            return False
        if url.startswith('data:') and not re.match(
            r'^data:image/(?:png|jpeg|gif|webp|svg\+xml)', url, re.IGNORECASE
        ):
            return False
        return True
    
    @classmethod
    def escape_html(cls, text: str) -> str:
        return (text
            .replace('&', '&amp;')
            .replace('<', '&lt;')
            .replace('>', '&gt;')
            .replace('"', '&quot;')
            .replace("'", '&#39;'))
    
    @classmethod
    def has_dangerous_content(cls, html: str) -> bool:
        for pattern in cls.DANGEROUS_PATTERNS:
            if pattern.search(html):
                return True
        return False
    
    @classmethod
    def check_input(cls, user_input: str) -> Dict:
        """Quick check for XSS attempts in user input"""
        threats = []
        
        xss_patterns = {
            'script_tag': re.compile(r'<\s*script', re.IGNORECASE),
            'event_handler': re.compile(r'\bon\w+\s*=\s*["\']?[^"\']* ["\']?', re.IGNORECASE),
            'javascript_url': re.compile(r'javascript\s*:', re.IGNORECASE),
            'vbscript_url': re.compile(r'vbscript\s*:', re.IGNORECASE),
            'data_url': re.compile(r'data\s*:text/html', re.IGNORECASE),
            'eval': re.compile(r'\beval\s*\(', re.IGNORECASE),
            'expression': re.compile(r'expression\s*\(', re.IGNORECASE),
            'iframe': re.compile(r'<\s*iframe', re.IGNORECASE),
            'object': re.compile(r'<\s*object', re.IGNORECASE),
        }
        
        for threat_name, pattern in xss_patterns.items():
            m = pattern.search(user_input)
            if m:
                threats.append({
                    'type': threat_name,
                    'match': m.group(0),
                    'position': m.start(),
                })
        
        return {
            'safe': len(threats) == 0,
            'threats': threats,
            'threat_count': len(threats),
        }


# ทดสอบ XSS Detection
xss_attempts = [
    '<script>alert("XSS")</script>',
    '<img src=x onerror=alert(1)>',
    '<a href="javascript:alert(1)">Click</a>',
    '<div onmouseover="steal()">Hover me</div>',
    '"><script>document.cookie</script>',
    '<iframe src="//evil.com/xss">',
    'Hello World',  # safe
    '<p>Normal <b>text</b></p>',  # safe
]

print("\nXSS Detection:")
print("=" * 60)
for payload in xss_attempts:
    result = HTMLSanitizer.check_input(payload)
    icon = 'DANGER' if not result['safe'] else 'SAFE'
    if result['threats']:
        threat_names = ', '.join(t['type'] for t in result['threats'])
        print(f"  [{icon}] {payload[:45]!r}")
        print(f"       Threats: {threat_names}")
    else:
        print(f"  [{icon}] {payload[:45]!r}")
```

---

## 17.7 Extract Structured Data from HTML

```python
import re
from typing import List, Dict

class TableExtractor:
    """Extract tables from HTML"""
    
    TABLE_PATTERN = re.compile(r'<table\b[^>]*>(.*?)</table>', re.IGNORECASE | re.DOTALL)
    ROW_PATTERN = re.compile(r'<tr\b[^>]*>(.*?)</tr>', re.IGNORECASE | re.DOTALL)
    CELL_PATTERN = re.compile(r'<t[dh]\b[^>]*>(.*?)</t[dh]>', re.IGNORECASE | re.DOTALL)
    TAG_PATTERN = re.compile(r'<[^>]+>')
    
    @classmethod
    def extract_tables(cls, html: str) -> List[List[List[str]]]:
        tables = []
        for table_m in cls.TABLE_PATTERN.finditer(html):
            table_html = table_m.group(1)
            table = []
            for row_m in cls.ROW_PATTERN.finditer(table_html):
                row = []
                for cell_m in cls.CELL_PATTERN.finditer(row_m.group(1)):
                    text = cls.TAG_PATTERN.sub('', cell_m.group(1))
                    text = re.sub(r'\s+', ' ', text).strip()
                    row.append(text)
                if row:
                    table.append(row)
            if table:
                tables.append(table)
        return tables


class ListExtractor:
    """Extract lists from HTML"""
    
    LIST_PATTERN = re.compile(r'<(?P<type>u|o)l\b[^>]*>(.*?)</(?P=type)l>', re.IGNORECASE | re.DOTALL)
    ITEM_PATTERN = re.compile(r'<li\b[^>]*>(.*?)</li>', re.IGNORECASE | re.DOTALL)
    TAG_PATTERN = re.compile(r'<[^>]+>')
    
    @classmethod
    def extract_lists(cls, html: str) -> List[Dict]:
        lists = []
        for m in cls.LIST_PATTERN.finditer(html):
            list_type = 'unordered' if m.group('type').lower() == 'u' else 'ordered'
            items = []
            for item_m in cls.ITEM_PATTERN.finditer(m.group(2)):
                text = cls.TAG_PATTERN.sub('', item_m.group(1))
                text = re.sub(r'\s+', ' ', text).strip()
                if text:
                    items.append(text)
            if items:
                lists.append({'type': list_type, 'items': items})
        return lists


# ทดสอบ
html_with_table = """
<html><body>
<h2>ผลการทดสอบ</h2>
<table>
  <tr><th>ชื่อ</th><th>คะแนน</th><th>เกรด</th></tr>
  <tr><td>สมชาย</td><td>85</td><td>A</td></tr>
  <tr><td>สมหญิง</td><td>72</td><td>B</td></tr>
  <tr><td>สมศรี</td><td>91</td><td>A+</td></tr>
</table>

<ul>
  <li>Python</li>
  <li>JavaScript</li>
  <li>PHP</li>
</ul>

<ol>
  <li>เรียน Regex พื้นฐาน</li>
  <li>ฝึกทำแบบฝึกหัด</li>
  <li>สร้างโปรเจกต์จริง</li>
</ol>
</body></html>
"""

print("\nTable Extraction:")
tables = TableExtractor.extract_tables(html_with_table)
for i, table in enumerate(tables):
    print(f"\n  Table {i+1}:")
    for row in table:
        print(f"    {' | '.join(row)}")

print("\nList Extraction:")
lists = ListExtractor.extract_lists(html_with_table)
for lst in lists:
    print(f"\n  {lst['type'].capitalize()} list:")
    for item in lst['items']:
        marker = 'o' if lst['type'] == 'unordered' else '-'
        print(f"    {marker} {item}")
```

---

## 17.8 JavaScript HTML Parsing

```javascript
// JavaScript HTML Parsing with Regex
// Note: In browser environments, prefer DOMParser
// These patterns are for Node.js/non-browser contexts

const HTMLParser = {
    stripTags(html) {
        return html
            .replace(/<(script|style)[^>]*>.*?<\/\1>/gis, '')
            .replace(/<!--.*?-->/gs, '')
            .replace(/<[^>]+>/g, '')
            .replace(/&amp;/g, '&')
            .replace(/&lt;/g, '<')
            .replace(/&gt;/g, '>')
            .replace(/&quot;/g, '"')
            .replace(/&#39;/g, "'")
            .replace(/&nbsp;/g, ' ')
            .replace(/\s+/g, ' ')
            .trim();
    },
    
    extractLinks(html) {
        const pattern = /<a\b[^>]*\bhref\s*=\s*["']([^"']+)["'][^>]*>/gi;
        const links = [];
        let match;
        while ((match = pattern.exec(html)) !== null) {
            const url = match[1].trim();
            if (url && !url.startsWith('#') && !url.startsWith('javascript:')) {
                links.push(url);
            }
        }
        return [...new Set(links)];
    },
    
    extractImages(html) {
        const pattern = /<img\b([^>]*)>/gi;
        const attrPattern = /\b(\w[\w-]*)\s*=\s*["']([^"']*)["']/gi;
        const images = [];
        let imgMatch;
        
        while ((imgMatch = pattern.exec(html)) !== null) {
            const attrs = {};
            let attrMatch;
            attrPattern.lastIndex = 0;
            
            while ((attrMatch = attrPattern.exec(imgMatch[1])) !== null) {
                attrs[attrMatch[1].toLowerCase()] = attrMatch[2];
            }
            
            if (attrs.src) images.push(attrs);
        }
        return images;
    },
    
    hasXSS(input) {
        const patterns = [
            /<\s*script/i,
            /\bon\w+\s*=/i,
            /javascript\s*:/i,
            /vbscript\s*:/i,
            /expression\s*\(/i,
        ];
        return patterns.some(p => p.test(input));
    },
    
    escapeHTML(text) {
        return text
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#39;');
    }
};

// ทดสอบ
const sampleHTML = `
<div>
    <a href="https://example.com">Link 1</a>
    <a href="/about">Link 2</a>
    <img src="/logo.png" alt="Logo">
    <p>Hello <b>World</b> &amp; more</p>
</div>
`;

console.log('Strip:', HTMLParser.stripTags(sampleHTML));
console.log('Links:', HTMLParser.extractLinks(sampleHTML));
console.log('Images:', HTMLParser.extractImages(sampleHTML));

const xssTests = [
    '<script>alert(1)</script>',
    '<img onerror=alert(1)>',
    'Safe text',
];
xssTests.forEach(t => {
    console.log(`${HTMLParser.hasXSS(t) ? 'DANGER' : 'SAFE'} ${t.slice(0, 30)}`);
});
```

---

## 17.9 สรุป Part 17

```
Pattern สำคัญสำหรับ HTML:
strip tags:    <[^>]+>
HTML comment:  <!--.*?-->   (DOTALL mode)
Script block:  <script[^>]*>.*?</script>  (IGNORECASE+DOTALL)
href attr:     href\s*=\s*["']([^"']+)["']
src attr:      src\s*=\s*["']([^"']+)["']
Any attr:      \b(\w[\w-]*)\s*=\s*["']([^"']*)["']
Event handler: \bon\w+\s*=  (XSS detection)
JS URL:        javascript\s*:  (XSS detection)
```

**คำเตือน**: Regex ไม่ใช่ทางเลือกที่ดีที่สุดสำหรับ HTML parsing ที่ซับซ้อน  
ใช้ `BeautifulSoup` / `lxml` ใน Python หรือ `DOMParser` ใน Browser  
ใช้ Regex สำหรับ: strip tags, extract URLs, sanitize, XSS detection  

---

*[← Part 16: IP Addresses](part-16-ip-addresses.md) | [→ Part 18: Log File Parsing](part-18-log-parsing.md)*
