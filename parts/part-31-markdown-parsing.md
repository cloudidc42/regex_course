# Part 31: Markdown Parsing — การแยกวิเคราะห์ Markdown

> **ระดับ:** กลาง | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-30

---

## 31.1 Markdown Elements

```python
import re
from typing import List, Dict, Optional

class MarkdownParser:
    """Parse Markdown elements ด้วย regex"""
    
    # Headings
    HEADING = re.compile(r'^(#{1,6})\s+(.+?)(?:\s+#+)?\s*$', re.MULTILINE)
    
    # Bold/Italic
    BOLD    = re.compile(r'\*\*(.+?)\*\*|__(.+?)__')
    ITALIC  = re.compile(r'\*(?!\*)(.+?)(?<!\*)\*|_(?!_)(.+?)(?<!_)_')
    BOLD_ITALIC = re.compile(r'\*\*\*(.+?)\*\*\*|___(.+?)___')
    STRIKETHROUGH = re.compile(r'~~(.+?)~~')
    CODE_INLINE = re.compile(r'`([^`]+)`')
    
    # Links and images
    LINK  = re.compile(r'\[([^\]]+)\]\(([^)]+)\)')
    IMAGE = re.compile(r'!\[([^\]]*)\]\(([^)]+)\)')
    AUTOLINK = re.compile(r'<(https?://[^>]+)>')
    REF_LINK = re.compile(r'\[([^\]]+)\]\[([^\]]*)\]')
    
    # Code blocks
    FENCED_CODE = re.compile(
        r'^```(\w+)?\n(.*?)^```',
        re.MULTILINE | re.DOTALL
    )
    INDENTED_CODE = re.compile(r'^(?: {4}|\t)(.+)$', re.MULTILINE)
    
    # Lists
    UNORDERED_LIST = re.compile(r'^[ \t]*[-*+]\s+(.+)$', re.MULTILINE)
    ORDERED_LIST   = re.compile(r'^[ \t]*\d+\.\s+(.+)$', re.MULTILINE)
    
    # Blockquote
    BLOCKQUOTE = re.compile(r'^>\s*(.+)$', re.MULTILINE)
    
    # Horizontal rule
    HR = re.compile(r'^(?:\*{3,}|-{3,}|_{3,})\s*$', re.MULTILINE)
    
    # Tables
    TABLE_ROW = re.compile(r'^\|(.+)\|$', re.MULTILINE)
    TABLE_SEP = re.compile(r'^\|(?::?-+:?\|)+$', re.MULTILINE)
    
    @classmethod
    def extract_headings(cls, text: str) -> List[Dict]:
        headings = []
        for m in cls.HEADING.finditer(text):
            level = len(m.group(1))
            title = m.group(2).strip()
            anchor = re.sub(r'[^a-z0-9\-]', '', title.lower().replace(' ', '-'))
            headings.append({'level': level, 'title': title, 'anchor': anchor})
        return headings
    
    @classmethod
    def extract_links(cls, text: str) -> List[Dict]:
        links = []
        for m in cls.LINK.finditer(text):
            if not text[max(0, m.start()-1)] == '!':
                links.append({'text': m.group(1), 'url': m.group(2)})
        for m in cls.AUTOLINK.finditer(text):
            links.append({'text': m.group(1), 'url': m.group(1)})
        return links
    
    @classmethod
    def extract_code_blocks(cls, text: str) -> List[Dict]:
        blocks = []
        for m in cls.FENCED_CODE.finditer(text):
            blocks.append({
                'language': m.group(1) or 'text',
                'code': m.group(2),
                'lines': m.group(2).count('\n') + 1,
            })
        return blocks
    
    @classmethod
    def extract_table(cls, text: str) -> Optional[List[List[str]]]:
        """ดึง table data"""
        rows = cls.TABLE_ROW.findall(text)
        if len(rows) < 2:
            return None
        
        result = []
        for i, row in enumerate(rows):
            # Skip separator row
            if re.match(r'^[ :\-|]+$', row):
                continue
            cells = [c.strip() for c in row.split('|')]
            result.append(cells)
        
        return result if len(result) > 1 else None
    
    @classmethod
    def to_plain_text(cls, text: str) -> str:
        """แปลง Markdown เป็น plain text"""
        text = cls.FENCED_CODE.sub(r'\2', text)
        text = cls.IMAGE.sub(r'[\1]', text)
        text = cls.LINK.sub(r'\1', text)
        text = cls.BOLD_ITALIC.sub(r'\1\2', text)
        text = cls.BOLD.sub(r'\1\2', text)
        text = cls.ITALIC.sub(r'\1\2', text)
        text = cls.STRIKETHROUGH.sub(r'\1', text)
        text = cls.CODE_INLINE.sub(r'\1', text)
        text = cls.HEADING.sub(r'\2', text)
        text = cls.BLOCKQUOTE.sub(r'\1', text)
        text = cls.HR.sub('---', text)
        return text.strip()
    
    @classmethod
    def generate_toc(cls, text: str) -> str:
        """สร้าง Table of Contents"""
        headings = cls.extract_headings(text)
        if not headings:
            return ''
        
        lines = ['## Table of Contents\n']
        for h in headings:
            indent = '  ' * (h['level'] - 1)
            lines.append(f"{indent}- [{h['title']}](#{h['anchor']})")
        
        return '\n'.join(lines)


# ทดสอบ
markdown_text = """
# Python Regex Tutorial

## 1. Introduction

Regular expressions (**regex**) are *powerful* patterns for text matching.

## 2. Basic Patterns

### 2.1 Character Classes

- `[abc]` matches a, b, or c
- `[^abc]` matches anything except a, b, c
- `\d` matches digits

```python
import re
pattern = re.compile(r'\d+')
matches = pattern.findall("Price: 100 baht")
print(matches)  # ['100']
```

### 2.2 Quantifiers

| Symbol | Meaning | Example |
|--------|---------|----------|
| `*` | 0 or more | `a*` |
| `+` | 1 or more | `a+` |
| `?` | 0 or 1 | `a?` |

See [Python docs](https://docs.python.org/3/library/re.html) for more info.

---

## 3. Advanced Topics

> Always test your regex with real data before deploying.

"""

parser = MarkdownParser

print("Markdown Analysis:")
print("=" * 60)

headings = parser.extract_headings(markdown_text)
print(f"\nHeadings ({len(headings)}):")
for h in headings:
    print(f"  {'#' * h['level']} {h['title']}")

links = parser.extract_links(markdown_text)
print(f"\nLinks ({len(links)}):")
for l in links:
    print(f"  [{l['text']}]({l['url']})")

code_blocks = parser.extract_code_blocks(markdown_text)
print(f"\nCode blocks ({len(code_blocks)}):")
for b in code_blocks:
    print(f"  language={b['language']}, lines={b['lines']}")

toc = parser.generate_toc(markdown_text)
print(f"\nGenerated TOC:\n{toc}")
```

---

## 31.2 Markdown to HTML (Basic)

```python
import re

class MarkdownToHTML:
    """แปลง Markdown เป็น HTML แบบ basic"""
    
    def convert(self, text: str) -> str:
        # Code blocks (ต้องทำก่อน เพราะมี special chars ข้างใน)
        text = re.sub(
            r'```(\w+)?\n(.*?)```',
            lambda m: f'<pre><code class="language-{m.group(1) or "text"}">{self._escape(m.group(2))}</code></pre>',
            text, flags=re.DOTALL
        )
        
        # Inline code
        text = re.sub(r'`([^`]+)`', lambda m: f'<code>{self._escape(m.group(1))}</code>', text)
        
        # Headings
        for i in range(6, 0, -1):
            text = re.sub(
                rf'^{"#"*i}\s+(.+?)$',
                lambda m, level=i: f'<h{level}>{m.group(1)}</h{level}>',
                text, flags=re.MULTILINE
            )
        
        # Bold/Italic
        text = re.sub(r'\*\*\*(.+?)\*\*\*', r'<strong><em>\1</em></strong>', text)
        text = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', text)
        text = re.sub(r'\*(.+?)\*', r'<em>\1</em>', text)
        text = re.sub(r'~~(.+?)~~', r'<del>\1</del>', text)
        
        # Links and images
        text = re.sub(r'!\[([^\]]*)\]\(([^)]+)\)', r'<img alt="\1" src="\2">', text)
        text = re.sub(r'\[([^\]]+)\]\(([^)]+)\)', r'<a href="\2">\1</a>', text)
        
        # Blockquote
        text = re.sub(r'^>\s*(.+)$', r'<blockquote>\1</blockquote>', text, flags=re.MULTILINE)
        
        # Horizontal rule
        text = re.sub(r'^(?:\*{3,}|-{3,}|_{3,})\s*$', '<hr>', text, flags=re.MULTILINE)
        
        # Unordered lists
        text = re.sub(r'^[-*+]\s+(.+)$', r'<li>\1</li>', text, flags=re.MULTILINE)
        
        # Ordered lists
        text = re.sub(r'^\d+\.\s+(.+)$', r'<li>\1</li>', text, flags=re.MULTILINE)
        
        # Paragraphs (แปลง double newline เป็น paragraph)
        paragraphs = re.split(r'\n{2,}', text.strip())
        result = []
        for p in paragraphs:
            p = p.strip()
            if p and not re.match(r'^<(?:h[1-6]|pre|blockquote|ul|ol|li|hr)', p):
                p = f'<p>{p}</p>'
            result.append(p)
        
        return '\n'.join(result)
    
    def _escape(self, text: str) -> str:
        return (text
            .replace('&', '&amp;')
            .replace('<', '&lt;')
            .replace('>', '&gt;'))


md = MarkdownToHTML()
sample = """# Hello World

This is **bold** and *italic* text with `inline code`.

[Visit Python](https://python.org)

- Item one
- Item two
- Item three

---

> A wise quote goes here.
"""

html = md.convert(sample)
print("Markdown to HTML:")
print("=" * 60)
print(html)
```

---

## 31.3 สรุป Part 31

```
Markdown Patterns:
Heading:     ^(#{1,6})\s+(.+?)$  (MULTILINE)
Bold:        \*\*(.+?)\*\* or __(.+?)__
Italic:      \*(.+?)\* or _(.+?)_
Code inline: `([^`]+)`
Code block:  ```lang\n(.*?)```  (DOTALL|MULTILINE)
Link:        \[([^\]]+)\]\(([^)]+)\)
Image:       !\[([^\]]*)\]\(([^)]+)\)
List item:   ^[-*+]\s+(.+)$  (MULTILINE)
Blockquote:  ^>\s*(.+)$  (MULTILINE)
HR:          ^(?:\*{3,}|-{3,}|_{3,})\s*$

Tips:
- Process code blocks FIRST (preserve content)
- Handle bold-italic before bold/italic separately
- Use re.DOTALL for multi-line code blocks
- re.MULTILINE for line-anchored patterns (^/$)
```

---

*[← Part 30: Date & Time](part-30-datetime-patterns.md) | [→ Part 32: Code Analysis](part-32-code-analysis.md)*
