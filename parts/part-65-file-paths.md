# Part 65: File & Path Processing with Regex

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-64

---

## 65.1 File Path Parsing

```python
import re
from typing import Dict, Optional, List

print("File & Path Processing with Regex:")
print("=" * 60)

print("\n1. Cross-platform path parsing:")

UNIFIED_PATH = re.compile(
    r'^(?:(?P<drive>[A-Za-z]):[/\\])?'
    r'(?P<dirs>(?:[^/\\:\0]+[/\\])*)'
    r'(?P<basename>[^/\\:\0]*)$'
)

FILE_EXT    = re.compile(r'\.([^./\\]+)$')
HIDDEN_FILE = re.compile(r'(?:^|[/\\])\.(?!\.)[^/\\]+$')


def parse_path(path: str) -> Dict:
    m = UNIFIED_PATH.match(path.replace('\\', '/'))
    if not m:
        return {}
    d  = m.groupdict()
    bn = d.get('basename', '')
    ext_m = FILE_EXT.search(bn)
    stem  = bn[:ext_m.start()] if ext_m else bn
    return {
        'drive':    d.get('drive'),
        'dirs':     d.get('dirs', '').rstrip('/'),
        'basename': bn,
        'stem':     stem,
        'ext':      ext_m.group(1) if ext_m else None,
        'hidden':   bool(HIDDEN_FILE.search(path)),
        'absolute': path.startswith('/') or bool(d.get('drive')),
    }


paths = [
    '/home/user/documents/report.pdf',
    'C:/Users/Alice/Documents/script.py',
    '../config/settings.json',
    '.env',
    'data/raw/2024/sales.csv',
    '/etc/nginx/nginx.conf',
    'README.md',
    '/home/user/.bashrc',
]

print(f"\n   {'Path':<45} {'Stem':<18} {'Ext':<8} {'Abs':<6} {'Hidden'}")
print(f"   {'-'*45} {'-'*18} {'-'*8} {'-'*6} {'-'*6}")
for path in paths:
    p = parse_path(path)
    print(f"   {path:<45} {(p.get('stem') or '-'):<18} {(p.get('ext') or '-'):<8} "
          f"{str(p.get('absolute', False)):<6} {str(p.get('hidden', False))}")
```

---

## 65.2 Glob Pattern Matching

```python
import re
from typing import List

print("\nGlob Pattern Matching:")
print("=" * 60)

def glob_to_regex(pattern: str) -> re.Pattern:
    result = []
    i = 0
    while i < len(pattern):
        c = pattern[i]
        if c == '*':
            if i + 1 < len(pattern) and pattern[i+1] == '*':
                result.append('.*')
                i += 2
                if i < len(pattern) and pattern[i] == '/':
                    i += 1
            else:
                result.append('[^/]*')
        elif c == '?':
            result.append('[^/]')
        elif c == '[':
            j = pattern.find(']', i)
            if j == -1:
                result.append(re.escape(c))
            else:
                result.append(pattern[i:j+1])
                i = j
        else:
            result.append(re.escape(c))
        i += 1
    return re.compile('^' + ''.join(result) + '$')


test_files = [
    'src/main.py', 'src/utils/helpers.py', 'src/utils/math/calc.py',
    'tests/test_main.py', 'docs/README.md', 'config.json',
    'requirements.txt', '.gitignore', 'src/__init__.py',
]

glob_patterns = [
    ('*.py',        'Python files in root'),
    ('src/**/*.py', 'All .py under src/'),
    ('**/*.py',     'All .py anywhere'),
    ('tests/*.py',  'Python tests'),
    ('[!.]*',       'Files not starting with .'),
]

for pattern, desc in glob_patterns:
    try:
        rx = glob_to_regex(pattern)
        matches = [f for f in test_files if rx.match(f)]
        print(f"\n   Pattern: {pattern!r}  ({desc})")
        print(f"   Regex:   {rx.pattern}")
        print(f"   Matches: {matches}")
    except re.error as e:
        print(f"\n   Pattern: {pattern!r} \u2014 error: {e}")
```

---

## 65.3 File Content Pattern Matching

```python
import re
from typing import List, Dict

print("\nFile Content Pattern Matching:")
print("=" * 60)

SHEBANG    = re.compile(r'^#!(.+)$')
ENCODING   = re.compile(r'#.*?coding[:=]\s*([-\w.]+)')
VERSION    = re.compile(r'^__version__\s*=\s*[\'\"]([\'\"]+)[\'\"]', re.MULTILINE)
AUTHOR     = re.compile(r'^__author__\s*=\s*[\'\"]([\'\"]+)[\'\"]', re.MULTILINE)

VERSION2   = re.compile(r'__version__\s*=\s*["\']([^"\']+)["\']')
AUTHOR2    = re.compile(r'__author__\s*=\s*["\']([^"\']+)["\']')
LICENSE_RE = re.compile(r'__license__\s*=\s*["\']([^"\']+)["\']')


def extract_file_metadata(source: str) -> Dict:
    meta = {}
    m = SHEBANG.match(source)
    if m:
        meta['shebang'] = m.group(1).strip()
    m = ENCODING.search(source[:200])
    if m:
        meta['encoding'] = m.group(1)
    m = VERSION2.search(source)
    if m:
        meta['version'] = m.group(1)
    m = AUTHOR2.search(source)
    if m:
        meta['author'] = m.group(1)
    m = LICENSE_RE.search(source)
    if m:
        meta['license'] = m.group(1)
    return meta


sample_file = '''#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""My awesome package."""

__version__ = "2.3.1"
__author__ = "Alice Smith"
__license__ = "MIT"

import re
'''

meta = extract_file_metadata(sample_file)
print(f"\n   File metadata:")
for k, v in meta.items():
    print(f"   {k:<12}: {v}")


print("\n\n2. Security scanning:")

GREP_PATTERNS = {
    'ip_addr':  re.compile(r'\b(?:\d{1,3}\.){3}\d{1,3}\b'),
    'url':      re.compile(r'https?://[^\s<>"\']+'),
    'email':    re.compile(r'\b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b'),
    'secret':   re.compile(r'(?i)(?:password|passwd|secret|token)\s*[=:]\s*["\']?([^\s"\',;]+)'),
    'api_key':  re.compile(r'(?i)api[_-]?key\s*[=:]\s*["\']?([a-zA-Z0-9_\-]{20,})["\']?'),
    'todo':     re.compile(r'(?i)#\s*(TODO|FIXME|HACK|BUG)\s*:?\s*(.+)'),
}

def grep_file(content: str, patterns: Dict) -> Dict:
    results = {}
    for name, pattern in patterns.items():
        found = []
        for i, line in enumerate(content.splitlines(), 1):
            for m in pattern.finditer(line):
                found.append({'line': i, 'text': line.strip()[:60], 'match': m.group(0)[:40]})
        if found:
            results[name] = found
    return results


code_sample = '''
API_KEY = "sk-abcdef1234567890abcdef1234567890"
DB_PASSWORD = "my_super_secret_pass"
SERVER_URL = "https://api.example.com/v2"
BACKUP_IP = "192.168.1.100"
admin_email = "admin@example.com"
# TODO: Remove hardcoded credentials before deploy
'''

grep_results = grep_file(code_sample, GREP_PATTERNS)
print(f"\n   Security scan results:")
for category, findings in grep_results.items():
    print(f"\n   [{category.upper()}]")
    for f in findings[:2]:
        print(f"   Line {f['line']}: {f['text'][:60]}")
```

---

## 65.4 .gitignore Pattern Converter

```python
import re
from typing import Optional, List

print("\n.gitignore Pattern Processing:")
print("=" * 60)

def gitignore_to_regex(pattern: str) -> Optional[re.Pattern]:
    if not pattern or pattern.startswith('#'):
        return None
    if pattern.startswith('!'):
        pattern = pattern[1:]
    pattern = pattern.strip()
    result  = []
    anchored = pattern.startswith('/')
    if anchored:
        result.append('^')
        pattern = pattern[1:]
    else:
        result.append('(?:^|.*/')
        result.append(')')
    i = 0
    while i < len(pattern):
        c = pattern[i]
        if c == '*':
            if i + 1 < len(pattern) and pattern[i+1] == '*':
                result.append('.*')
                i += 2
                if i < len(pattern) and pattern[i] == '/':
                    i += 1
            else:
                result.append('[^/]*')
        elif c == '?':
            result.append('[^/]')
        elif c == '[':
            j = pattern.find(']', i)
            result.append(pattern[i:j+1] if j != -1 else re.escape(c))
            i = j if j != -1 else i
        else:
            result.append(re.escape(c))
        i += 1
    result.append('(?:/.*)?$')
    try:
        return re.compile(''.join(result))
    except re.error:
        return None


test_rules_files = [
    ('__pycache__/', '__pycache__/module.cpython-311.pyc'),
    ('*.pyc',       'src/main.pyc'),
    ('.env',        '.env'),
    ('dist/',       'dist/package-1.0.tar.gz'),
    ('*.log',       'server.log'),
    ('/config/local.py', 'config/local.py'),
    ('README.md',   'README.md'),
]

print(f"\n   {'Rule':<25} {'File':<40} {'Ignored'}")
print(f"   {'-'*25} {'-'*40} {'-'*8}")
for rule, filepath in test_rules_files:
    rx = gitignore_to_regex(rule)
    if rx:
        ignored = bool(rx.match(filepath))
        print(f"   {rule:<25} {filepath:<40} {'yes' if ignored else 'no'}")
```

---

## 65.5 สรุป Part 65

```
File & Path Regex Patterns:

1. Path components:
   Unix:    ^(/)?((dir/)*)?(basename)(\.ext)?$
   Windows: ^([A-Za-z]:[/\\])?((dir\\)*)?(name)(\.ext)?$
   Hidden:  (?:^|[/\\])\.(?!\.)[^/\\]+$

2. Glob \u2192 Regex conversion:
   *   \u2192 [^/]*  (within one directory)
   **  \u2192 .*     (any depth)
   ?   \u2192 [^/]   (one char, no slash)
   [x] \u2192 [x]    (character class)
   Escape everything else with re.escape()

3. File metadata:
   Shebang:  ^#!(.+)$
   Encoding: #.*?coding[:=]\s*([-\w.]+)
   Version:  __version__\s*=\s*['"]([\'"]+)['"]  

4. Security scanning:
   Hardcoded secrets: (password|token)\s*[=:]\s*[^\s]+
   API keys:          api[_-]?key\s*[=:]\s*[a-zA-Z0-9]{20,}

5. .gitignore rules:
   Anchored /prefix \u2192 ^pattern
   Unanchored \u2192 (?:^|.*/)pattern
   Trailing / \u2192 directory match only
```

---

*[\u2190 Part 64: Unicode & Internationalization](part-64-unicode.md) | [\u2192 Part 66: Configuration & Formats](part-66-config-formats.md)*
