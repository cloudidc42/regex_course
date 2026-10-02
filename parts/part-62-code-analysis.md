# Part 62: Code Analysis & Source Parsing with Regex

> **ระดับ:** สูง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-61

---

## 62.1 Python Source Code Analysis

```python
import re
from typing import Dict, List, Optional

print("Code Analysis & Source Parsing:")
print("=" * 60)

print("\n1. Python function/class extraction:")

PY_FUNCTION  = re.compile(
    r'^(?P<indent>\s*)(?P<async>async\s+)?def\s+(?P<name>[a-zA-Z_]\w*)'
    r'\s*\((?P<params>[^)]*)\)\s*(?:->(?P<return>[^:]+))?:\s*$',
    re.MULTILINE
)

PY_CLASS = re.compile(
    r'^(?P<indent>\s*)class\s+(?P<name>[a-zA-Z_]\w*)'
    r'\s*(?:\((?P<bases>[^)]*)\))?\s*:\s*$',
    re.MULTILINE
)

PY_DECORATOR = re.compile(
    r'^(?P<indent>\s*)@(?P<name>[a-zA-Z_][\w.]*(?:\([^)]*\))?)\s*$',
    re.MULTILINE
)

PY_IMPORT = re.compile(
    r'^(?:from\s+(?P<module>[\w.]+)\s+import\s+(?P<names>[^\n]+)|'
    r'import\s+(?P<plain>[\w.,\s]+))$',
    re.MULTILINE
)


def analyze_python(source: str) -> Dict:
    result = {
        'functions':  [],
        'classes':    [],
        'imports':    [],
        'decorators': [],
    }
    for m in PY_FUNCTION.finditer(source):
        params_raw = m.group('params').strip()
        params     = [p.strip().split(':')[0].strip().lstrip('*')
                      for p in params_raw.split(',') if p.strip()] if params_raw else []
        result['functions'].append({
            'name':    m.group('name'),
            'async':   bool(m.group('async')),
            'params':  params,
            'returns': m.group('return').strip() if m.group('return') else None,
            'indent':  len(m.group('indent')),
        })
    for m in PY_CLASS.finditer(source):
        result['classes'].append({
            'name':  m.group('name'),
            'bases': [b.strip() for b in m.group('bases').split(',')] if m.group('bases') else [],
        })
    for m in PY_IMPORT.finditer(source):
        if m.group('module'):
            result['imports'].append(f"from {m.group('module')} import {m.group('names').strip()}")
        elif m.group('plain'):
            result['imports'].append(f"import {m.group('plain').strip()}")
    for m in PY_DECORATOR.finditer(source):
        result['decorators'].append(m.group('name'))
    return result


sample_py = '''
import re
from typing import List, Optional
from dataclasses import dataclass

@dataclass
class User:
    """A user model."""
    name: str
    email: str

@staticmethod
def validate_email(email: str) -> bool:
    """Validate email format."""
    return bool(re.match(r"[^@]+@[^@]+\\.[^@]+", email))

async def fetch_user(user_id: int, include_posts: bool = False) -> Optional[User]:
    """Fetch user from database."""
    pass

class UserRepository:
    def get(self, id: int) -> User: ...
    def save(self, user: User) -> bool: ...
    async def delete(self, id: int) -> None: ...
'''

info = analyze_python(sample_py)
print(f"\n   Classes:    {[c['name'] for c in info['classes']]}")
print(f"   Imports:    {info['imports'][:3]}")
print(f"   Decorators: {info['decorators']}")
print(f"\n   Functions:")
print(f"   {'Name':<20} {'Async':<8} {'Params':<30} {'Return'}")
print(f"   {'-'*20} {'-'*8} {'-'*30} {'-'*12}")
for fn in info['functions']:
    params = ', '.join(fn['params'][:4])
    print(f"   {fn['name']:<20} {str(fn['async']):<8} {params:<30} {fn['returns'] or '-'}")
```

---

## 62.2 JavaScript/TypeScript Parsing

```python
import re
from typing import Dict, List

print("\nJavaScript/TypeScript Analysis:")
print("=" * 60)

JS_FUNCTION = re.compile(
    r'(?:export\s+)?(?:async\s+)?function\s*(?P<name>[a-zA-Z_$]\w*)\s*'
    r'\((?P<params>[^)]*)\)\s*(?::\s*(?P<return>[^{]+))?\s*\{',
    re.MULTILINE
)

JS_ARROW = re.compile(
    r'(?:const|let|var)\s+(?P<name>[a-zA-Z_$]\w*)\s*=\s*'
    r'(?:async\s+)?\((?P<params>[^)]*)\)\s*(?::\s*[^=>\s]+\s*)?=>\s*(?:\{|[^;\n])',
    re.MULTILINE
)

JS_CLASS = re.compile(
    r'(?:export\s+)?class\s+(?P<name>[A-Z][a-zA-Z_$]\w*)'
    r'(?:\s+extends\s+(?P<parent>[A-Z][a-zA-Z_$]\w*))?\s*\{',
    re.MULTILINE
)

JS_IMPORT = re.compile(
    r"import\s+(?:(?P<default>\w+)\s*,?\s*)?"
    r"(?:\{\s*(?P<named>[^}]+)\s*\})?"
    r"\s+from\s+['\"](?P<module>[^'\"]+)['\"]",
    re.MULTILINE
)

TS_INTERFACE = re.compile(
    r'(?:export\s+)?interface\s+(?P<name>[A-Z][a-zA-Z_$]\w*)'
    r'(?:\s+extends\s+(?P<parent>[A-Z][a-zA-Z_$]\w*))?\s*\{',
    re.MULTILINE
)

TS_TYPE = re.compile(
    r'(?:export\s+)?type\s+(?P<name>[A-Z][a-zA-Z_$]\w*)\s*=\s*(?P<def>[^;]+)',
    re.MULTILINE
)


def analyze_js(source: str) -> Dict:
    return {
        'functions':  [m.groupdict() for m in JS_FUNCTION.finditer(source)],
        'arrows':     [m.groupdict() for m in JS_ARROW.finditer(source)],
        'classes':    [m.groupdict() for m in JS_CLASS.finditer(source)],
        'imports':    [m.groupdict() for m in JS_IMPORT.finditer(source)],
        'interfaces': [m.groupdict() for m in TS_INTERFACE.finditer(source)],
        'types':      [m.groupdict() for m in TS_TYPE.finditer(source)],
    }


ts_code = '''
import React, { useState, useEffect } from "react";
import { User, ApiResponse } from "./types";
import axios from "axios";

interface UserProfile extends User {
    posts: Post[];
    settings: Settings;
}

type ApiStatus = "loading" | "success" | "error";

class UserService {
    private baseUrl: string = "/api";

    async getUser(id: number): Promise<User> {
        return axios.get(`${this.baseUrl}/users/${id}`);
    }
}

export async function fetchUsers(page: number, limit: number): Promise<User[]> {
    const response = await axios.get("/api/users", { params: { page, limit } });
    return response.data;
}

const formatName = (user: User): string => `${user.firstName} ${user.lastName}`;

export const createUser = async (data: Partial<User>): Promise<User> => {
    return axios.post("/api/users", data);
};
'''

info = analyze_js(ts_code)
print(f"\n   Functions:  {[f['name'] for f in info['functions']]}")
print(f"   Arrows:     {[a['name'] for a in info['arrows']]}")
print(f"   Classes:    {[c['name'] for c in info['classes']]}")
print(f"   Interfaces: {[i['name'] for i in info['interfaces']]}")
print(f"   Types:      {[t['name'] for t in info['types']]}")
print(f"   Imports:")
for imp in info['imports']:
    named = imp.get('named', '').strip()[:30] if imp.get('named') else '-'
    print(f"     {imp['module']:<30} named: {named}")
```

---

## 62.3 Code Quality & Smell Detection

```python
import re
from typing import List, Dict

print("\nCode Quality & Smell Detection:")
print("=" * 60)

TODO_RE = re.compile(
    r'#\s*(?P<type>TODO|FIXME|HACK|XXX|NOTE|BUG|OPTIMIZE|REVIEW)\s*:?\s*(?P<text>[^\n]+)',
    re.IGNORECASE
)

LONG_LINE = re.compile(r'^.{90,}$', re.MULTILINE)

PRINT_DEBUG = re.compile(r'\bprint\s*\(', re.MULTILINE)

BARE_EXCEPT = re.compile(r'\bexcept\s*:', re.MULTILINE)

MUTABLE_DEFAULT = re.compile(
    r'def\s+\w+\s*\([^)]*=\s*(?:\[\s*\]|\{\s*\})',
    re.MULTILINE
)

PASS_ONLY = re.compile(r'(?:def|class)\s+\w[^:]+:\s*\n\s+pass\s*$', re.MULTILINE)


def find_code_smells(source: str) -> Dict[str, List]:
    smells = {}

    todos = [{'type': m.group('type').upper(), 'text': m.group('text').strip(),
              'line': source[:m.start()].count('\n') + 1}
             for m in TODO_RE.finditer(source)]
    if todos: smells['todos'] = todos

    long_lines = [{'line': i+1, 'length': len(ln)}
                  for i, ln in enumerate(source.splitlines()) if len(ln) > 89]
    if long_lines: smells['long_lines'] = long_lines

    if PRINT_DEBUG.search(source):
        smells['debug_prints'] = [{'line': source[:m.start()].count('\n')+1}
                                   for m in PRINT_DEBUG.finditer(source)]
    if BARE_EXCEPT.search(source):
        smells['bare_excepts'] = [{'line': source[:m.start()].count('\n')+1}
                                   for m in BARE_EXCEPT.finditer(source)]
    if MUTABLE_DEFAULT.search(source):
        smells['mutable_defaults'] = True
    if PASS_ONLY.search(source):
        smells['empty_functions'] = True

    return smells


smelly_code = '''
def process_data(items=[], config={}):  # mutable default!
    # TODO: Add validation
    # FIXME: This is very slow for large inputs
    x = 42  # magic number
    print(f"Processing {len(items)} items")  # debug print
    result = []
    for item in items:
        try:
            result.append(item * x)
        except:  # bare except!
            pass
    return result

def empty_function():
    pass
'''

smells = find_code_smells(smelly_code)
print(f"\n   Code smells found:")
for smell_type, findings in smells.items():
    if isinstance(findings, list):
        print(f"\n   {smell_type}:")
        for f in findings[:3]:
            print(f"     Line {f.get('line', '?')}: {f.get('type', '')} {f.get('text', '')[:50]}")
    else:
        print(f"   {smell_type}: detected")
```

---

## 62.4 Dependency & Import Graph Analysis

```python
import re
from collections import defaultdict
from typing import Dict, List, Set

print("\nDependency Analysis:")
print("=" * 60)

PYTHON_IMPORT   = re.compile(r'^import\s+([\w,\s.]+)$', re.MULTILINE)
PYTHON_FROM     = re.compile(r'^from\s+([\w.]+)\s+import\s+(.+)$', re.MULTILINE)
STDLIB_MODULES  = re.compile(
    r'^(?:re|os|sys|io|abc|ast|copy|json|math|time|uuid|enum|'
    r'typing|types|pathlib|datetime|functools|itertools|collections|'
    r'contextlib|dataclasses|threading|multiprocessing|asyncio|'
    r'unittest|logging|hashlib|hmac|secrets|base64|urllib|http|'
    r'socket|ssl|email|csv|sqlite3|argparse|shutil|tempfile)$'
)


def analyze_imports(source: str) -> Dict:
    deps   = {'stdlib': set(), 'third_party': set(), 'local': set()}
    for m in PYTHON_IMPORT.finditer(source):
        for mod in m.group(1).split(','):
            mod = mod.strip().split(' ')[0].split('.')[0]
            if STDLIB_MODULES.match(mod):
                deps['stdlib'].add(mod)
            elif not mod.startswith('.'):
                deps['third_party'].add(mod)
    for m in PYTHON_FROM.finditer(source):
        mod = m.group(1).split('.')[0]
        if mod.startswith('.') or m.group(1).startswith('.'):
            deps['local'].add(m.group(1))
        elif STDLIB_MODULES.match(mod):
            deps['stdlib'].add(mod)
        else:
            deps['third_party'].add(mod)
    return {k: sorted(v) for k, v in deps.items()}


multi_file_imports = '''
import re
import os
import sys
import json
import logging
from typing import Dict, List, Optional
from datetime import datetime
from collections import Counter, defaultdict
from dataclasses import dataclass, field

import requests
import fastapi
from fastapi import HTTPException, Depends
from sqlalchemy import Column, Integer, String
from pydantic import BaseModel, validator
import redis
import celery
from prometheus_client import Counter as PCounter

from .models import User, Post
from .utils import validate_email
from ..config import Settings
'''

analysis = analyze_imports(multi_file_imports)
print(f"\n   Import Analysis:")
print(f"\n   Standard Library ({len(analysis['stdlib'])}):\n   {analysis['stdlib']}")
print(f"\n   Third-party ({len(analysis['third_party'])}):\n   {analysis['third_party']}")
print(f"\n   Local ({len(analysis['local'])}):\n   {analysis['local']}")
```

---

## 62.5 สรุป Part 62

```
Code Analysis Regex Patterns:

1. Python parsing:
   Function: ^(\s*)(async\s+)?def\s+(\w+)\s*\(([^)]*)\)...
   Class:    ^(\s*)class\s+(\w+)(?:\(([^)]*)\))?:
   Import:   ^(?:from\s+([\w.]+)\s+import|import\s+)(.+)$
   Decorator: ^(\s*)@([\w.]+(?:\([^)]*\))?)

2. JavaScript/TypeScript:
   Function: (?:async\s+)?function\s+(\w+)\s*\(([^)]*)\)
   Arrow:    (?:const|let)\s+(\w+)\s*=\s*(?:async\s+)?\(([^)]*)\)=>
   Class:    class\s+([A-Z]\w*)(?:\s+extends\s+([A-Z]\w*))?
   Interface: interface\s+([A-Z]\w*)(?:\s+extends\s+([A-Z]\w*))?

3. Code smells:
   TODO/FIXME:      #\s*(TODO|FIXME|HACK)\s*:?\s*(.+)
   Bare except:     \bexcept\s*:
   Mutable default: def\s+\w+\([^)]*=\s*(?:\[\]|\{\})
   Debug print:     \bprint\s*\(

4. Import classification:
   Stdlib:       known stdlib module names
   Third-party:  not starting with . or in stdlib
   Local:        starting with . or known local path
```

---

*[← Part 61: Text Mining & NLP](part-61-text-mining.md) | [→ Part 63: Advanced Pattern Matching](part-63-advanced-patterns.md)*
