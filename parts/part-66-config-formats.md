# Part 66: Configuration & Format Parsing

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-65

---

## 66.1 YAML-like Parsing with Regex

```python
import re
from typing import Dict, Any

print("Configuration & Format Parsing:")
print("=" * 60)

print("\n1. YAML-like flat key-value parsing:")

YAML_KEY_VAL = re.compile(r'^(?P<indent>\s*)(?P<key>[a-zA-Z_][\w.-]*):\s*(?P<value>.*)$', re.MULTILINE)
YAML_COMMENT = re.compile(r'^\s*#.*$', re.MULTILINE)


def parse_yaml_flat(text: str) -> Dict[str, Any]:
    text   = YAML_COMMENT.sub('', text)
    result = {}
    for m in YAML_KEY_VAL.finditer(text):
        if len(m.group('indent')) > 0:
            continue
        key   = m.group('key')
        value = m.group('value').strip()
        if value.startswith('"') and value.endswith('"'):
            value = value[1:-1]
        elif value.startswith("'") and value.endswith("'"):
            value = value[1:-1]
        if value.lower() in ('null', 'none', '~'):
            result[key] = None
        elif value.lower() in ('true', 'yes', 'on'):
            result[key] = True
        elif value.lower() in ('false', 'no', 'off'):
            result[key] = False
        elif re.match(r'^-?\d+$', value):
            result[key] = int(value)
        elif re.match(r'^-?\d+\.\d+$', value):
            result[key] = float(value)
        else:
            result[key] = value
    return result


yaml_sample = """
# Application configuration
name: my-app
version: "2.3.1"
debug: false
port: 8080
host: "0.0.0.0"
max_connections: 100
timeout: 30.5
log_level: INFO
secret_key: null
"""

parsed = parse_yaml_flat(yaml_sample)
print(f"\n   Parsed YAML (flat):")
for k, v in parsed.items():
    print(f"   {k:<20} = {v!r:<30} ({type(v).__name__})")
```

---

## 66.2 TOML Parsing

```python
import re
from typing import Dict, Any

print("\nTOML Parsing:")
print("=" * 60)

TOML_SECTION = re.compile(r'^\[(?P<name>[^\[\]]+)\]$', re.MULTILINE)
TOML_KV      = re.compile(r'^(?P<key>[\w.-]+)\s*=\s*(?P<value>.+)$', re.MULTILINE)
TOML_STRING  = re.compile(r'^"((?:[^"\\]|\\.)*)"$')
TOML_INT     = re.compile(r'^-?\d[\d_]*$')
TOML_FLOAT   = re.compile(r'^-?\d[\d_]*\.\d[\d_]*(?:[eE][+-]?\d+)?$')
TOML_BOOL    = re.compile(r'^(?:true|false)$')
TOML_ARRAY   = re.compile(r'^\[(.+)\]$')
TOML_COMMENT = re.compile(r'#[^\n]*$', re.MULTILINE)


def parse_toml_value(raw: str) -> Any:
    raw = raw.strip()
    m = TOML_STRING.match(raw)
    if m:
        return m.group(1).replace('\\n', '\n').replace('\\t', '\t')
    if TOML_BOOL.match(raw):
        return raw == 'true'
    if TOML_INT.match(raw):
        return int(raw.replace('_', ''))
    if TOML_FLOAT.match(raw):
        return float(raw.replace('_', ''))
    m = TOML_ARRAY.match(raw)
    if m:
        items = [s.strip() for s in m.group(1).split(',') if s.strip()]
        return [parse_toml_value(item) for item in items]
    return raw


def parse_toml(text: str) -> Dict:
    text   = TOML_COMMENT.sub('', text)
    result = {}
    current_section = result
    for line in text.splitlines():
        line = line.strip()
        if not line:
            continue
        m = TOML_SECTION.match(line)
        if m:
            parts   = m.group('name').strip().split('.')
            current = result
            for part in parts:
                current.setdefault(part, {})
                current = current[part]
            current_section = current
            continue
        m = TOML_KV.match(line)
        if m:
            current_section[m.group('key').strip()] = parse_toml_value(m.group('value').strip())
    return result


toml_sample = """
[app]
name = "my-service"
version = "1.2.3"
debug = false
port = 8080

[database]
host = "localhost"
port = 5432
pool_size = 10
timeout = 30.5

[features]
enabled = ["auth", "cache", "metrics"]
max_retries = 3
"""

config = parse_toml(toml_sample)
print(f"\n   Parsed TOML: {list(config.keys())}")
for section, values in config.items():
    print(f"\n   [{section}]")
    for k, v in values.items():
        print(f"   {k:<15} = {v!r}  ({type(v).__name__})")
```

---

## 66.3 Dockerfile Parsing

```python
import re
from typing import Dict, List

print("\nDockerfile Parsing:")
print("=" * 60)

DOCKERFILE_INSTRUCTION = re.compile(
    r'^(?P<instr>FROM|RUN|CMD|LABEL|EXPOSE|ENV|ADD|COPY|ENTRYPOINT|'
    r'VOLUME|USER|WORKDIR|ARG|ONBUILD|STOPSIGNAL|HEALTHCHECK|SHELL)'
    r'\s+(?P<args>.+)$',
    re.MULTILINE | re.IGNORECASE
)

DOCKER_FROM  = re.compile(
    r'^FROM\s+(?P<image>[^\s:@]+)(?::(?P<tag>[^\s@]+))?'
    r'(?:@(?P<digest>\S+))?\s*(?:AS\s+(?P<stage>\S+))?$',
    re.IGNORECASE | re.MULTILINE
)
DOCKER_ENV   = re.compile(r'^ENV\s+(?P<key>[A-Z_][A-Z0-9_]*)\s+(?P<value>.+)$', re.MULTILINE | re.IGNORECASE)
DOCKER_EXPOSE = re.compile(r'^EXPOSE\s+(?P<ports>[\d\s/]+)$', re.MULTILINE | re.IGNORECASE)
DOCKER_COPY  = re.compile(
    r'^COPY\s+(?:--from=(?P<from_stage>\S+)\s+)?(?P<src>.+?)\s+(?P<dest>\S+)$',
    re.MULTILINE | re.IGNORECASE
)


def parse_dockerfile(content: str) -> Dict:
    result = {'from': [], 'env': {}, 'ports': [], 'copy': [], 'layers': 0}
    for m in DOCKER_FROM.finditer(content):
        result['from'].append({'image': m.group('image'), 'tag': m.group('tag') or 'latest', 'stage': m.group('stage')})
    for m in DOCKER_ENV.finditer(content):
        result['env'][m.group('key')] = m.group('value').strip()
    for m in DOCKER_EXPOSE.finditer(content):
        result['ports'].extend(m.group('ports').split())
    for m in DOCKER_COPY.finditer(content):
        result['copy'].append({'src': m.group('src'), 'dest': m.group('dest'), 'from': m.group('from_stage')})
    instructions = DOCKERFILE_INSTRUCTION.findall(content)
    result['layers'] = sum(1 for instr, _ in instructions if instr.upper() in ('RUN', 'COPY', 'ADD'))
    return result


dockerfile = """
FROM python:3.11-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS development
COPY . .
EXPOSE 8000 8080
CMD ["python", "-m", "uvicorn", "main:app", "--reload"]

FROM base AS production
COPY --from=development /app/src ./src
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app"]
"""

docker_info = parse_dockerfile(dockerfile)
print(f"\n   Dockerfile analysis:")
print(f"   Stages:       {[s['stage'] for s in docker_info['from'] if s['stage']]}")
print(f"   Base images:  {[f\"{f['image']}:{f['tag']}\" for f in docker_info['from']]}")
print(f"   Env vars:     {docker_info['env']}")
print(f"   Exposed ports:{docker_info['ports']}")
print(f"   Total layers: {docker_info['layers']}")
```

---

## 66.4 GitHub Actions Pattern Analysis

```python
import re
from typing import Dict, List

print("\nGitHub Actions Analysis:")
print("=" * 60)

GH_STEP_NAME = re.compile(r'^\s+-?\s+name:\s+(?P<name>.+)$', re.MULTILINE)
GH_USES = re.compile(r'^\s+uses:\s+(?P<action>[^\s@]+)@(?P<version>[^\s#]+)', re.MULTILINE)
GH_SECRET = re.compile(r'\$\{\{\s*secrets\.(?P<name>[A-Z_][A-Z0-9_]*)\s*\}\}')
GH_EXPR   = re.compile(r'\$\{\{\s*(?P<expr>[^}]+)\s*\}\}')


def analyze_github_actions(content: str) -> Dict:
    return {
        'steps':   [m.group('name').strip() for m in GH_STEP_NAME.finditer(content)],
        'actions': [f"{m.group('action')}@{m.group('version')}" for m in GH_USES.finditer(content)],
        'secrets': sorted(set(m.group('name') for m in GH_SECRET.finditer(content))),
        'exprs':   sorted(set(m.group('expr').strip() for m in GH_EXPR.finditer(content)))[:6],
    }


github_workflow = """
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest --cov=src
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}

  deploy:
    needs: test
    if: ${{ github.ref == 'refs/heads/main' }}
    steps:
      - name: Deploy to production
        run: ./deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
"""

ci_info = analyze_github_actions(github_workflow)
print(f"\n   GitHub Actions analysis:")
print(f"   Steps:   {ci_info['steps']}")
print(f"   Actions: {ci_info['actions']}")
print(f"   Secrets: {ci_info['secrets']}")
print(f"   Expressions: {ci_info['exprs']}")
```

---

## 66.5 สรุป Part 66

```
Config & Format Parsing Regex:

1. YAML key-value:
   ^(\s*)([a-zA-Z_][\w.-]*):\s*(.*)$
   - indent level = len(match.group(1))
   - coerce: int, float, bool, null

2. TOML:
   Section: ^\[([^\[\]]+)\]$
   KV:      ^([\w.-]+)\s*=\s*(.+)$
   Array:   ^\[(.+)\]$ → split by comma

3. Dockerfile:
   Instruction: ^(FROM|RUN|CMD|ENV|COPY|EXPOSE...)
   FROM:  image:tag@digest AS stage
   ENV:   KEY value
   COPY:  --from=stage src dest

4. GitHub Actions:
   Step name:  ^\s+- name:\s+(.+)$
   Uses:       ^\s+uses:\s+([^@]+)@(.+)$
   Secrets:    \$\{\{ secrets\.([A-Z_]+) \}\}
   Expressions: \$\{\{ (.+?) \}\}

5. Common config patterns:
   Comments: #[^\n]* (Python/YAML/TOML)
   Sections: ^\[([^\[\]]+)\]$ (INI/TOML)
   Quoted:   "([^"\\]|\\.)*" (JSON/TOML)
```

---

*[← Part 65: File & Path Processing](part-65-file-paths.md) | [→ Part 67: Web Scraping & HTML Parsing](part-67-web-scraping.md)*
