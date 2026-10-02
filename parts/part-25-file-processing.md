# Part 25: File Processing — การประมวลผลไฟล์

> **ระดับ:** กลาง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-24

---

## 25.1 ประมวลผลไฟล์ข้อความขนาดใหญ่

```python
import re
from pathlib import Path
from typing import Iterator, List, Dict

def search_file_generator(filepath: str, pattern: re.Pattern) -> Iterator[dict]:
    """ค้นหา pattern ในไฟล์ขนาดใหญ่โดยใช้ generator"""
    with open(filepath, 'r', encoding='utf-8', errors='replace') as f:
        for lineno, line in enumerate(f, 1):
            for m in pattern.finditer(line):
                yield {
                    'line': lineno,
                    'col': m.start() + 1,
                    'match': m.group(0),
                    'groups': m.groups(),
                    'content': line.rstrip(),
                }

def grep_file(filepath: str, pattern: str, flags: int = 0, context: int = 0) -> List[dict]:
    """grep-like ค้นหาในไฟล์"""
    p = re.compile(pattern, flags)
    results = []
    
    with open(filepath, 'r', encoding='utf-8', errors='replace') as f:
        lines = f.readlines()
    
    for i, line in enumerate(lines):
        if p.search(line):
            result = {
                'lineno': i + 1,
                'line': line.rstrip(),
                'context_before': [l.rstrip() for l in lines[max(0, i-context):i]],
                'context_after': [l.rstrip() for l in lines[i+1:i+1+context]],
            }
            results.append(result)
    
    return results

def count_pattern_occurrences(filepath: str, patterns: Dict[str, str]) -> Dict[str, int]:
    """นับจำนวน occurrences ของหลาย pattern"""
    compiled = {name: re.compile(p) for name, p in patterns.items()}
    counts = {name: 0 for name in patterns}
    
    with open(filepath, 'r', encoding='utf-8', errors='replace') as f:
        for line in f:
            for name, p in compiled.items():
                counts[name] += len(p.findall(line))
    
    return counts


# ตัวอย่าง: สร้างไฟล์ทดสอบ
import tempfile, os

sample_content = """2026-10-02 14:30:00 INFO  Server started on port 8080
2026-10-02 14:30:01 DEBUG Database connected: postgresql://localhost/app
2026-10-02 14:30:02 INFO  User admin@example.com logged in from 192.168.1.100
2026-10-02 14:30:03 ERROR Failed to process request: Connection timeout
2026-10-02 14:30:04 WARN  High memory usage: 85% (3.4GB/4GB)
2026-10-02 14:30:05 INFO  Request GET /api/users HTTP/1.1 200 1234ms
2026-10-02 14:30:06 ERROR Database query failed: SELECT * FROM users WHERE id=?
2026-10-02 14:30:07 INFO  Cache hit: user_123
2026-10-02 14:30:08 DEBUG SQL: UPDATE sessions SET last_seen=NOW() WHERE id=123
"""

with tempfile.NamedTemporaryFile(mode='w', suffix='.log', delete=False, encoding='utf-8') as f:
    f.write(sample_content)
    tmpfile = f.name

print("File Pattern Matching:")
print("=" * 60)

# ค้นหา ERROR lines
errors = grep_file(tmpfile, r'\bERROR\b')
print(f"\nErrors found: {len(errors)}")
for e in errors:
    print(f"  Line {e['lineno']}: {e['line']}")

# นับ pattern types
counts = count_pattern_occurrences(tmpfile, {
    'info':    r'\bINFO\b',
    'error':   r'\bERROR\b',
    'warning': r'\bWARN\b',
    'debug':   r'\bDEBUG\b',
    'emails':  r'[\w.+]+@[\w.]+\.[a-z]{2,}',
    'ips':     r'\b(?:\d{1,3}\.){3}\d{1,3}\b',
})
print(f"\nOccurrence counts:")
for name, count in counts.items():
    print(f"  {name:10} {count}")

os.unlink(tmpfile)
```

---

## 25.2 Batch File Processing

```python
import re
from pathlib import Path
from typing import List, Dict, Tuple
import os, tempfile

class FileProcessor:
    """ประมวลผลไฟล์หลายไฟล์ด้วย regex"""
    
    def __init__(self, patterns: Dict[str, re.Pattern]):
        self.patterns = patterns
    
    def scan_directory(self, dirpath: str, glob_pattern: str = '**/*') -> Dict:
        """สแกน directory ค้นหา patterns"""
        results = []
        root = Path(dirpath)
        
        for filepath in root.glob(glob_pattern):
            if filepath.is_file():
                file_results = self._scan_file(filepath)
                if file_results['matches']:
                    results.append(file_results)
        
        return {
            'files_scanned': len(list(root.glob(glob_pattern))),
            'files_with_matches': len(results),
            'results': results,
        }
    
    def _scan_file(self, filepath: Path) -> Dict:
        matches = []
        try:
            with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
                for lineno, line in enumerate(f, 1):
                    for pattern_name, pattern in self.patterns.items():
                        for m in pattern.finditer(line):
                            matches.append({
                                'pattern': pattern_name,
                                'line': lineno,
                                'match': m.group(0),
                            })
        except (IOError, PermissionError):
            pass
        
        return {'file': str(filepath), 'matches': matches}
    
    def replace_in_file(self, filepath: str, replacements: List[Tuple[str, str]], 
                         dry_run: bool = True) -> Dict:
        """แทนที่ patterns ในไฟล์"""
        with open(filepath, 'r', encoding='utf-8') as f:
            original = f.read()
        
        modified = original
        changes = []
        
        for pattern_str, replacement in replacements:
            p = re.compile(pattern_str)
            new_text, count = p.subn(replacement, modified)
            if count:
                changes.append({'pattern': pattern_str, 'replacement': replacement, 'count': count})
                modified = new_text
        
        if not dry_run and changes:
            with open(filepath, 'w', encoding='utf-8') as f:
                f.write(modified)
        
        return {
            'file': filepath,
            'changes': changes,
            'total_changes': sum(c['count'] for c in changes),
            'dry_run': dry_run,
        }


# ตัวอย่าง
print("Batch File Processing:")

processor = FileProcessor({
    'email': re.compile(r'[\w.+]+@[\w.]+\.[a-z]{2,}'),
    'ip':    re.compile(r'\b(?:\d{1,3}\.){3}\d{1,3}\b'),
    'error': re.compile(r'\berror\b', re.IGNORECASE),
})

# สร้างไฟล์ทดสอบ
tmpdir = tempfile.mkdtemp()
for i, content in enumerate([
    "admin@example.com logged in from 192.168.1.1\nERROR: connection failed",
    "No special patterns here\njust normal text",
    "user@test.org visited from 10.0.0.5\n",
]):
    with open(f"{tmpdir}/file{i+1}.txt", 'w') as f:
        f.write(content)

result = processor.scan_directory(tmpdir, '**/*.txt')
print(f"\n  Files scanned: {result['files_scanned']}")
print(f"  Files with matches: {result['files_with_matches']}")
for r in result['results']:
    fname = os.path.basename(r['file'])
    print(f"\n  {fname}:")
    for m in r['matches']:
        print(f"    Line {m['line']} [{m['pattern']}]: {m['match']}")

import shutil
shutil.rmtree(tmpdir)
```

---

## 25.3 CSV Processing

```python
import re
from typing import List, Dict, Iterator

class CSVProcessor:
    """ประมวลผล CSV ด้วย regex"""
    
    # CSV field: quoted หรือ unquoted
    CSV_FIELD = re.compile(
        r'"((?:[^"]|"")*)'    # quoted field
        r'|([^,\n]*)'          # unquoted field
    )
    
    @classmethod
    def parse_line(cls, line: str) -> List[str]:
        fields = []
        for m in cls.CSV_FIELD.finditer(line):
            if m.group(1) is not None:
                fields.append(m.group(1).replace('""', '"'))
            else:
                fields.append(m.group(2))
        # Remove trailing empty from last comma
        if fields and line.endswith(','):
            fields.append('')
        return fields
    
    @classmethod
    def filter_rows(cls, filepath: str, column: int, pattern: str, 
                     has_header: bool = True) -> Iterator[List[str]]:
        """กรอง rows ที่ column ตรงกับ pattern"""
        p = re.compile(pattern)
        header = None
        
        with open(filepath, 'r', encoding='utf-8') as f:
            for i, line in enumerate(f):
                fields = cls.parse_line(line.strip())
                if i == 0 and has_header:
                    header = fields
                    continue
                if column < len(fields) and p.search(fields[column]):
                    yield fields
    
    @classmethod
    def transform_column(cls, data: List[str], column: int, 
                          pattern: str, replacement: str) -> List[str]:
        """แปลง column ด้วย regex"""
        result = list(data)
        if column < len(result):
            result[column] = re.sub(pattern, replacement, result[column])
        return result


# ตัวอย่าง: process CSV data
csv_data = """name,email,phone,salary
สมชาย ใจดี,somchai@example.com,089-123-4567,45000
สมหญิง รักเรียน,somying@test.org,091-234-5678,55000
อนุวัฒน์ พัฒนา,anuwat@company.co.th,02-345-6789,65000
สมศรี มีสุข,somsri@test.com,085-456-7890,48000
"""

import tempfile, os

with tempfile.NamedTemporaryFile(mode='w', suffix='.csv', delete=False, encoding='utf-8') as f:
    f.write(csv_data)
    csv_tmpfile = f.name

print("CSV Processing:")

# Parse CSV
lines = csv_data.strip().split('\n')
header = CSVProcessor.parse_line(lines[0])
print(f"\nHeader: {header}")

for line in lines[1:]:
    fields = CSVProcessor.parse_line(line)
    if fields:
        # Mask email
        masked_email = re.sub(r'^(.{2}).*(@.+)$', r'\1***\2', fields[1])
        # Format phone
        formatted_phone = re.sub(r'(\d{3})-(\d{3})-(\d{4})', r'(\1) \2-\3', fields[2])
        print(f"  {fields[0]:20} {masked_email:25} {formatted_phone}")

os.unlink(csv_tmpfile)
```

---

## 25.4 Config File Parser

```python
import re
from typing import Dict, Any
from pathlib import Path

class ConfigParser:
    """Parser สำหรับ config files ต่างๆ"""
    
    INI_SECTION  = re.compile(r'^\[(?P<section>[^\]]+)\]$')
    INI_KV       = re.compile(r'^(?P<key>[^=\s][^=]*?)\s*=\s*(?P<value>.+?)(?:\s*#.*)?$')
    COMMENT      = re.compile(r'^\s*[#;]')
    BLANK        = re.compile(r'^\s*$')
    YAML_KV      = re.compile(r'^(?P<indent>\s*)(?P<key>\w[\w-]*):\s*(?P<value>.*?)$')
    ENV_LINE     = re.compile(r'^(?P<key>[A-Z_][A-Z0-9_]*)=(?P<value>.*)$')
    
    @classmethod
    def parse_ini(cls, text: str) -> Dict:
        result = {}
        section = 'default'
        
        for line in text.split('\n'):
            if cls.COMMENT.match(line) or cls.BLANK.match(line):
                continue
            
            m = cls.INI_SECTION.match(line.strip())
            if m:
                section = m.group('section')
                result.setdefault(section, {})
                continue
            
            m = cls.INI_KV.match(line.strip())
            if m:
                key = m.group('key').strip()
                value = m.group('value').strip().strip('"\'')
                result.setdefault(section, {})[key] = value
        
        return result
    
    @classmethod
    def parse_env(cls, text: str) -> Dict:
        result = {}
        for line in text.split('\n'):
            if cls.COMMENT.match(line) or cls.BLANK.match(line):
                continue
            m = cls.ENV_LINE.match(line.strip())
            if m:
                key = m.group('key')
                value = m.group('value').strip('"\'')
                # Expand ${VAR} references
                value = re.sub(r'\$\{(\w+)\}', 
                               lambda mo: result.get(mo.group(1), mo.group(0)), 
                               value)
                result[key] = value
        return result


# ทดสอบ
ini_text = """
[server]
host = 0.0.0.0
port = 8080
debug = false  # enable debug mode

[database]
# PostgreSQL settings
host = localhost
port = 5432
name = myapp_db
user = appuser

[logging]
level = INFO
file = /var/log/app.log
"""

config = ConfigParser.parse_ini(ini_text)
print("INI Config:")
for section, values in config.items():
    print(f"\n  [{section}]")
    for k, v in values.items():
        print(f"    {k} = {v}")

env_text = """
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
APP_URL=http://${DB_HOST}:${DB_PORT}
SECRET_KEY="my-secret-key-123"
"""

env = ConfigParser.parse_env(env_text)
print("\n.env file:")
for k, v in env.items():
    print(f"  {k}={v}")
```

---

## 25.5 สรุป Part 25

```
File Processing Patterns:
grep line:    grep_file(path, pattern) → List[{lineno, line}]
Generator:    yield from file — ประหยัด memory
Batch scan:   scan_directory(path, glob) → matches per file
CSV parse:    "([^"]|"")*"|[^,\n]*  — handles quoted fields
INI parse:    ^\[([^\]]+)\]$  สำหรับ section
              ^([^=\s][^=]*?)\s*=\s*(.+?)$  สำหรับ key=value
ENV parse:    ^([A-Z_][A-Z0-9_]*)=(.*)$

Tips:
- ใช้ generator สำหรับ large files
- errors='replace' เพื่อหลีกเลี่ยง UnicodeDecodeError
- Path.glob() สำหรับ find files by pattern
- Dry run ก่อน replace จริง
```

---

*[← Part 24: Unicode](part-24-unicode.md) | [→ Part 26: Web Scraping](part-26-web-scraping.md)*
