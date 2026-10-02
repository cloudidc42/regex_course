# Part 32: Code Analysis Patterns — วิเคราะห์โค้ดด้วย Regex

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-31

---

## 32.1 Python Code Analyzer

```python
import re
from typing import List, Dict, Optional

class PythonCodeAnalyzer:
    """วิเคราะห์ Python source code ด้วย regex"""
    
    # Function/class definitions
    FUNCTION_DEF = re.compile(
        r'^(?P<indent>[ \t]*)(?P<async>async\s+)?def\s+(?P<name>[a-zA-Z_]\w*)'
        r'\((?P<params>[^)]*)\)\s*(?:->\s*(?P<return_type>[^:]+))?\s*:',
        re.MULTILINE
    )
    CLASS_DEF = re.compile(
        r'^(?P<indent>[ \t]*)class\s+(?P<name>[A-Z][a-zA-Z0-9_]*)'
        r'(?:\((?P<bases>[^)]*)\))?\s*:',
        re.MULTILINE
    )
    
    # Imports
    IMPORT = re.compile(
        r'^(?:from\s+(?P<from_module>[\w.]+)\s+)?'
        r'import\s+(?P<imports>[\w\s,.*]+)',
        re.MULTILINE
    )
    
    # String literals
    DOCSTRING = re.compile(
        r'(?:"""(.*?)"""|\'\'\'(.*?)\'\'\'')
        re.DOTALL
    )
    
    # Comments
    INLINE_COMMENT = re.compile(r'#\s*(.+)$', re.MULTILINE)
    TODO_COMMENT = re.compile(r'#\s*(?:TODO|FIXME|HACK|XXX|NOTE):\s*(.+)$', re.IGNORECASE | re.MULTILINE)
    
    # Variables / assignments
    ASSIGNMENT = re.compile(
        r'^(?P<indent>[ \t]*)(?P<name>[a-zA-Z_]\w*)\s*(?::\s*(?P<type>[^=]+?))?\s*=\s*(?P<value>.+)$',
        re.MULTILINE
    )
    
    # Decorators
    DECORATOR = re.compile(r'^[ \t]*@(?P<name>[\w.]+)(?:\((?P<args>[^)]*)\))?', re.MULTILINE)
    
    # Exception handling
    EXCEPTION = re.compile(r'\braise\s+(?P<exc>[A-Z][a-zA-Z0-9_]*)(?:\((?P<msg>[^)]*)\))?', re.MULTILINE)
    EXCEPT_CLAUSE = re.compile(r'\bexcept\s+(?P<exc>[A-Z][a-zA-Z0-9_, ]+?)(?:\s+as\s+\w+)?\s*:', re.MULTILINE)
    
    @classmethod
    def extract_functions(cls, code: str) -> List[Dict]:
        functions = []
        for m in cls.FUNCTION_DEF.finditer(code):
            indent_level = len(m.group('indent')) // 4
            is_method = indent_level > 0
            functions.append({
                'name': m.group('name'),
                'params': [p.strip() for p in m.group('params').split(',') if p.strip()],
                'return_type': (m.group('return_type') or '').strip() or None,
                'is_async': bool(m.group('async')),
                'is_method': is_method,
                'indent': indent_level,
            })
        return functions
    
    @classmethod
    def extract_classes(cls, code: str) -> List[Dict]:
        classes = []
        for m in cls.CLASS_DEF.finditer(code):
            bases_raw = m.group('bases') or ''
            bases = [b.strip() for b in bases_raw.split(',') if b.strip()]
            classes.append({
                'name': m.group('name'),
                'bases': bases,
            })
        return classes
    
    @classmethod
    def extract_imports(cls, code: str) -> List[Dict]:
        imports = []
        for m in cls.IMPORT.finditer(code):
            imports.append({
                'from': m.group('from_module'),
                'imports': [i.strip() for i in m.group('imports').split(',')],
            })
        return imports
    
    @classmethod
    def find_todos(cls, code: str) -> List[Dict]:
        todos = []
        for i, line in enumerate(code.split('\n'), 1):
            m = cls.TODO_COMMENT.search(line)
            if m:
                tag = re.search(r'#\s*(TODO|FIXME|HACK|XXX|NOTE)', line, re.IGNORECASE)
                todos.append({
                    'line': i,
                    'tag': tag.group(1).upper() if tag else 'TODO',
                    'text': m.group(1).strip(),
                })
        return todos
    
    @classmethod
    def analyze(cls, code: str) -> Dict:
        functions = cls.extract_functions(code)
        return {
            'functions': functions,
            'classes': cls.extract_classes(code),
            'imports': cls.extract_imports(code),
            'todos': cls.find_todos(code),
            'stats': {
                'total_functions': len(functions),
                'async_functions': sum(1 for f in functions if f['is_async']),
                'methods': sum(1 for f in functions if f['is_method']),
                'lines': code.count('\n') + 1,
            }
        }


# ทดสอบ
sample_code = '''
import re
from typing import List, Dict, Optional
from collections import Counter

# TODO: Add caching support
# FIXME: Memory leak in process_batch

class DataProcessor:
    """Process data with regex patterns."""
    
    DEFAULT_BATCH = 100
    
    def __init__(self, pattern: str, flags: int = 0):
        self.pattern = re.compile(pattern, flags)
        self.results: List[Dict] = []
    
    async def process_file(self, filepath: str) -> List[str]:
        """Process a file asynchronously."""
        matches = []
        # NOTE: Using generator for memory efficiency
        with open(filepath) as f:
            for line in f:
                if m := self.pattern.search(line):
                    matches.append(m.group())
        return matches
    
    @staticmethod
    def validate_pattern(pattern: str) -> bool:
        try:
            re.compile(pattern)
            return True
        except re.error:
            return False

def parse_csv_line(line: str, delimiter: str = ",") -> List[str]:
    """Parse a single CSV line."""
    return re.split(rf"(?<!\\){re.escape(delimiter)}", line)

def validate_email(email: str) -> bool:
    pattern = re.compile(r"^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$")
    return bool(pattern.match(email))
'''

analyzer = PythonCodeAnalyzer
result = analyzer.analyze(sample_code)

print("Python Code Analysis:")
print("=" * 60)
print(f"\nStats: {result['stats']}")

print(f"\nClasses ({len(result['classes'])}):")
for c in result['classes']:
    bases = f"({', '.join(c['bases'])})" if c['bases'] else ""
    print(f"  class {c['name']}{bases}")

print(f"\nFunctions ({len(result['functions'])}):")
for f in result['functions']:
    prefix = "async " if f['is_async'] else ""
    ret = f" -> {f['return_type']}" if f['return_type'] else ""
    method = " [method]" if f['is_method'] else ""
    print(f"  {prefix}def {f['name']}({', '.join(f['params'])}){ret}{method}")

print(f"\nImports ({len(result['imports'])}):")
for imp in result['imports']:
    if imp['from']:
        print(f"  from {imp['from']} import {', '.join(imp['imports'])}")
    else:
        print(f"  import {', '.join(imp['imports'])}")

print(f"\nTODOs/FIXMEs ({len(result['todos'])}):")
for t in result['todos']:
    print(f"  Line {t['line']:3d} [{t['tag']:5s}] {t['text']}")
```

---

## 32.2 JavaScript Code Analyzer

```python
import re
from typing import List, Dict

class JSCodeAnalyzer:
    """วิเคราะห์ JavaScript/TypeScript code"""
    
    # Function declarations
    FUNCTION_DECL = re.compile(
        r'(?P<async>async\s+)?function(?P<gen>\*)?\s+(?P<name>[a-zA-Z_$]\w*)'
        r'\s*\((?P<params>[^)]*)\)',
        re.MULTILINE
    )
    
    # Arrow functions assigned to const/let/var
    ARROW_FN = re.compile(
        r'(?:const|let|var)\s+(?P<name>[a-zA-Z_$]\w*)\s*=\s*'
        r'(?P<async>async\s+)?\((?P<params>[^)]*)\)\s*=>',
        re.MULTILINE
    )
    
    # ES6 imports
    ES6_IMPORT = re.compile(
        r"^import\s+"
        r"(?:(?P<default>\w+)(?:,\s*)?)?"
        r"(?:\{\s*(?P<named>[^}]+)\})?"
        r"(?:\*\s+as\s+(?P<namespace>\w+))?"
        r"\s*from\s*['\"](?P<module>[^'\"]+)['\"]",
        re.MULTILINE
    )
    
    # CommonJS requires
    CJS_REQUIRE = re.compile(
        r"(?:const|let|var)\s+(?:(?P<name>\w+)|\{(?P<destructure>[^}]+)\})"
        r"\s*=\s*require\(['\"](?P<module>[^'\"]+)['\"]\)",
        re.MULTILINE
    )
    
    # Console statements (debugging artifacts)
    CONSOLE_LOG = re.compile(r'\bconsole\.(log|warn|error|debug|info)\s*\(', re.MULTILINE)
    
    @classmethod
    def extract_functions(cls, code: str) -> List[Dict]:
        functions = []
        
        for m in cls.FUNCTION_DECL.finditer(code):
            functions.append({
                'name': m.group('name'),
                'type': 'function',
                'is_async': bool(m.group('async')),
                'is_generator': bool(m.group('gen')),
                'params': [p.strip() for p in m.group('params').split(',') if p.strip()],
            })
        
        for m in cls.ARROW_FN.finditer(code):
            functions.append({
                'name': m.group('name'),
                'type': 'arrow',
                'is_async': bool(m.group('async')),
                'params': [p.strip() for p in m.group('params').split(',') if p.strip()],
            })
        
        return functions
    
    @classmethod
    def extract_imports(cls, code: str) -> List[Dict]:
        imports = []
        
        for m in cls.ES6_IMPORT.finditer(code):
            named = []
            if m.group('named'):
                named = [n.strip() for n in m.group('named').split(',')]
            imports.append({
                'type': 'es6',
                'module': m.group('module'),
                'default': m.group('default'),
                'named': named,
                'namespace': m.group('namespace'),
            })
        
        for m in cls.CJS_REQUIRE.finditer(code):
            imports.append({
                'type': 'commonjs',
                'module': m.group('module'),
                'name': m.group('name'),
                'destructure': m.group('destructure'),
            })
        
        return imports
    
    @classmethod
    def find_debug_statements(cls, code: str) -> List[Dict]:
        debug = []
        for i, line in enumerate(code.split('\n'), 1):
            m = cls.CONSOLE_LOG.search(line)
            if m:
                debug.append({'line': i, 'type': m.group(1), 'content': line.strip()})
        return debug


# ทดสอบ
js_code = '''
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { validateEmail, formatDate } from '../utils/helpers';

const API_URL = process.env.REACT_APP_API_URL;

async function fetchUsers(page = 1) {
    const response = await axios.get(`${API_URL}/users?page=${page}`);
    return response.data;
}

const validateForm = async (formData) => {
    console.log('Validating form:', formData);
    const errors = {};
    if (!validateEmail(formData.email)) {
        errors.email = 'Invalid email';
    }
    return Object.keys(errors).length === 0;
};

function* generateIds(start = 1) {
    let id = start;
    while (true) yield id++;
}

class UserService {
    static async getUser(id) {
        console.warn('getUser called with id:', id);
        return await fetchUsers(1);
    }
    
    async updateProfile(userId, data) {
        const result = await axios.put(`${API_URL}/users/${userId}`, data);
        return result.data;
    }
}
'''

analyzer = JSCodeAnalyzer
functions = analyzer.extract_functions(js_code)
imports = analyzer.extract_imports(js_code)
debug = analyzer.find_debug_statements(js_code)

print("JavaScript Code Analysis:")
print("=" * 60)

print(f"\nFunctions/Arrows ({len(functions)}):")
for f in functions:
    async_tag = "async " if f['is_async'] else ""
    gen_tag = "* " if f.get('is_generator') else ""
    print(f"  [{f['type']:8}] {async_tag}{gen_tag}{f['name']}({', '.join(f['params'])})")

print(f"\nImports ({len(imports)}):")
for imp in imports:
    if imp['type'] == 'es6':
        parts = []
        if imp['default']: parts.append(imp['default'])
        if imp['named']: parts.append(f"{{{', '.join(imp['named'])}}}")
        print(f"  [ES6]  from '{imp['module']}': {', '.join(parts)}")
    else:
        name = imp['name'] or f"{{{imp['destructure']}}}"
        print(f"  [CJS]  require('{imp['module']}') -> {name}")

print(f"\nDebug Statements ({len(debug)}) — should remove before deploy:")
for d in debug:
    print(f"  Line {d['line']:3d} [console.{d['type']:5}] {d['content'][:60]}")
```

---

## 32.3 Security Code Scan

```python
import re
from typing import List, Dict

class SecurityScanner:
    """ตรวจหา security issues ใน source code"""
    
    # Hardcoded secrets
    HARDCODED_SECRET = re.compile(
        r'(?P<key>(?:password|passwd|secret|api_?key|token|auth|credential|private_key))'
        r'\s*[=:]\s*["\'](?P<value>[^"\']{8,})["\']',
        re.IGNORECASE
    )
    
    # SQL injection risks
    SQL_FORMAT_STRING = re.compile(
        r'(?:execute|query|cursor\.execute)\s*\(\s*'
        r'[f"\'%]*(?:SELECT|INSERT|UPDATE|DELETE)[^)]*%[sd]',
        re.IGNORECASE
    )
    
    # Command injection risks
    CMD_INJECTION = re.compile(
        r'(?:os\.system|subprocess\.call|subprocess\.run|eval|exec)\s*\('
        r'.*?(?:input|request\.|params\[|argv|user)',
        re.IGNORECASE | re.DOTALL
    )
    
    # Insecure random
    INSECURE_RANDOM = re.compile(
        r'\brandom\.(?:random|randint|choice|seed)\s*\(',
        re.MULTILINE
    )
    
    # Pickle deserialization
    PICKLE_LOAD = re.compile(r'\bpickle\.(?:load|loads)\s*\(', re.MULTILINE)
    
    # Weak hashing
    WEAK_HASH = re.compile(r'hashlib\.(?:md5|sha1)\s*\(', re.MULTILINE)
    
    SEVERITIES = {
        'hardcoded_secret': 'CRITICAL',
        'sql_format_string': 'HIGH',
        'cmd_injection': 'CRITICAL',
        'insecure_random': 'MEDIUM',
        'pickle_load': 'HIGH',
        'weak_hash': 'MEDIUM',
    }
    
    @classmethod
    def scan(cls, code: str, filename: str = '<code>') -> List[Dict]:
        findings = []
        lines = code.split('\n')
        
        checks = [
            ('hardcoded_secret', cls.HARDCODED_SECRET),
            ('sql_format_string', cls.SQL_FORMAT_STRING),
            ('cmd_injection', cls.CMD_INJECTION),
            ('insecure_random', cls.INSECURE_RANDOM),
            ('pickle_load', cls.PICKLE_LOAD),
            ('weak_hash', cls.WEAK_HASH),
        ]
        
        for check_name, pattern in checks:
            for m in pattern.finditer(code):
                line_num = code[:m.start()].count('\n') + 1
                line_content = lines[line_num - 1].strip() if line_num <= len(lines) else ''
                
                snippet = line_content
                if check_name == 'hardcoded_secret':
                    snippet = re.sub(r'([=:]\s*["\'])[^"\']+(["\'])', r'\1***\2', snippet)
                
                findings.append({
                    'file': filename,
                    'line': line_num,
                    'type': check_name,
                    'severity': cls.SEVERITIES[check_name],
                    'snippet': snippet[:80],
                })
        
        severity_order = {'CRITICAL': 0, 'HIGH': 1, 'MEDIUM': 2, 'LOW': 3}
        findings.sort(key=lambda f: severity_order.get(f['severity'], 99))
        
        return findings


# ทดสอบ
vulnerable_code = '''
import os, random, hashlib, pickle, sqlite3

DATABASE_PASSWORD = "super_secret_password_123"
API_KEY = "sk-1234567890abcdef"

def get_user(user_id, db_conn):
    query = "SELECT * FROM users WHERE id = %s" % user_id
    db_conn.execute(query)

def run_command(user_input):
    os.system("ls " + user_input)

def generate_token():
    return str(random.randint(100000, 999999))

def hash_password(password):
    return hashlib.md5(password.encode()).hexdigest()

def load_session(session_data):
    return pickle.loads(session_data)
'''

scanner = SecurityScanner
findings = scanner.scan(vulnerable_code, 'app.py')

print("Security Scan Results:")
print("=" * 60)

icons = {'CRITICAL': '[CRIT]', 'HIGH': '[HIGH]', 'MEDIUM': '[MED]', 'LOW': '[LOW]'}
for f in findings:
    icon = icons.get(f['severity'], '[???]')
    print(f"\n{icon} {f['type']}")
    print(f"   File: {f['file']}:{f['line']}")
    print(f"   Code: {f['snippet']}")

print(f"\nTotal: {len(findings)} findings")
```

---

## 32.4 สรุป Part 32

```
Code Analysis Patterns:
Python function:   ^(async\s+)?def\s+(\w+)\(([^)]*)\)(?:\s*->\s*([^:]+))?\s*:
Python class:      ^class\s+([A-Z]\w*)(?:\(([^)]*)\))?\s*:
Python import:     ^(?:from\s+([\w.]+)\s+)?import\s+([\w\s,.*]+)
JS function:       (async\s+)?function\*?\s+(\w+)\s*\(([^)]*)\)
JS arrow fn:       (?:const|let|var)\s+(\w+)\s*=\s*(async\s+)?\(([^)]*)\)\s*=>
ES6 import:        ^import\s+(?:(\w+),?\s*)?(?:\{([^}]+)\})?\s*from\s*['"]([^'"]+)['"]

Security patterns:
Hardcoded secret:  (password|secret|api_key)\s*[=:]\s*["']([^"']{8,})["']
SQL injection:     execute\(.*?%[sd] or SELECT.*?\+\s*(user|request)
CMD injection:     os\.system\(.*?(input|request|user)
Weak hash:         hashlib\.(md5|sha1)\(
Pickle load:       pickle\.loads?\(

Tips:
- แยก analysis กับ security scan ออกจากกัน
- Mask secrets ก่อน log หรือ report
- ใช้ DOTALL สำหรับ multi-line patterns
- เรียงผล findings ตาม severity
```

---

*[← Part 31: Markdown Parsing](part-31-markdown-parsing.md) | [→ Part 33: API Design Patterns](part-33-api-patterns.md)*
