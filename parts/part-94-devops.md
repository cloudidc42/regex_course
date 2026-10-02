# Part 94: Regex in DevOps & Infrastructure

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~110 นาที | **ข้อกำหนด:** Part 01-93

---

## 94.1 CI/CD Pipeline Patterns

```python
import re
from typing import Dict, List, Optional

print("Regex in DevOps & Infrastructure:")
print("=" * 60)

print("\n1. CI/CD pipeline patterns:")

# Git commit message validation (Conventional Commits spec)
CONVENTIONAL_COMMIT = re.compile(
    r'^(?P<type>feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)'
    r'(?:\((?P<scope>[^)]+)\))?'
    r'(?P<breaking>!)?'
    r':\s+'
    r'(?P<description>[^\n]{1,100})'
    r'(?:\n\n(?P<body>[\s\S]+?))?'
    r'(?:\n\n(?P<footer>(?:BREAKING CHANGE|[A-Za-z-]+#\d+)[\s\S]+?))?$'
)

def determine_bump(commit_msg: str) -> str:
    m = CONVENTIONAL_COMMIT.match(commit_msg)
    if not m:
        return 'none'
    if m.group('breaking') == '!' or 'BREAKING CHANGE' in (m.group('footer') or ''):
        return 'major'
    if m.group('type') == 'feat':
        return 'minor'
    if m.group('type') in ('fix', 'perf'):
        return 'patch'
    return 'none'


# Docker image tag patterns
DOCKER_TAG = re.compile(
    r'^(?P<registry>[a-zA-Z0-9.\-]+(?::\d+)?/)?'
    r'(?P<repository>[a-z0-9\-._/]+)'
    r'(?::(?P<tag>[a-zA-Z0-9.\-_]+))?'
    r'(?:@(?P<digest>sha256:[a-f0-9]{64}))?$'
)

# GitHub Actions workflow patterns
GA_USES = re.compile(r'uses:\s+(?P<action>[^\s@]+)@(?P<version>[^\s]+)')
GA_SECRET_REF = re.compile(r'\$\{\{\s*secrets\.(?P<name>[A-Z_][A-Z0-9_]*)\s*\}\}')

# Kubernetes resource name validation
K8S_NAME = re.compile(r'^[a-z0-9]([a-z0-9\-]{0,61}[a-z0-9])?$')


commits = [
    'feat(auth): add OAuth2 login support',
    'fix!: remove deprecated API endpoint',
    'docs: update README with installation steps',
    'feat(ui): redesign dashboard\n\nComplete UI overhaul\n\nBREAKING CHANGE: removed legacy dashboard',
    'chore: update dependencies',
    'not a conventional commit message',
    'fix(db): resolve connection pool exhaustion',
]

print(f"\n   Conventional commit analysis:")
for commit in commits:
    m = CONVENTIONAL_COMMIT.match(commit)
    bump = determine_bump(commit)
    first_line = commit.split('\n')[0]
    if m:
        print(f"\n   OK [{bump.upper():<5}] {first_line!r}")
        print(f"     type={m.group('type')!r}, scope={m.group('scope')!r}, breaking={'!' if m.group('breaking') else 'no'}")
    else:
        print(f"\n   INVALID: {first_line!r}")


docker_refs = [
    'nginx:latest',
    'my-registry.example.com:5000/team/app:v1.2.3',
    'python:3.11-slim',
]

print(f"\n   Docker image tag parsing:")
for ref in docker_refs:
    m = DOCKER_TAG.match(ref)
    if m:
        print(f"   {ref!r}")
        print(f"     registry={m.group('registry')!r}, repo={m.group('repository')!r}, tag={m.group('tag')!r}")
```

---

## 94.2 Infrastructure as Code Patterns

```python
import re
from typing import Dict, List, Tuple

print("\nInfrastructure as Code Patterns:")
print("=" * 60)

# Terraform resource patterns
TF_RESOURCE = re.compile(
    r'resource\s+"(?P<type>[^"]+)"\s+"(?P<name>[^"]+)"\s*\{'
)
TF_SENSITIVE = re.compile(
    r'(?:password|secret|token|key|credential)\s*=\s*"[^"]{3,}"',
    re.IGNORECASE
)

# Nginx config patterns
NGINX_SERVER_NAME = re.compile(r'server_name\s+([^;]+);')
NGINX_LOCATION = re.compile(r'location\s+(?P<modifier>[=~*^]+\s+)?(?P<path>[^\s{]+)\s*\{')
NGINX_PROXY_PASS = re.compile(r'proxy_pass\s+(?P<url>https?://[^;]+);')
NGINX_SSL_CERT = re.compile(r'ssl_certificate(?:_key)?\s+(?P<path>[^;]+);')


def analyze_terraform(tf_content: str) -> Dict:
    resources = TF_RESOURCE.findall(tf_content)
    sensitive = TF_SENSITIVE.findall(tf_content)
    return {
        'resources': [(t, n) for t, n in resources],
        'sensitive_values_found': len(sensitive),
        'resource_count': len(resources),
    }


def analyze_nginx_config(config: str) -> Dict:
    server_names = NGINX_SERVER_NAME.findall(config)
    locations    = [(m.group('modifier'), m.group('path')) for m in NGINX_LOCATION.finditer(config)]
    proxy_passes = NGINX_PROXY_PASS.findall(config)
    ssl_certs    = NGINX_SSL_CERT.findall(config)
    return {
        'server_names': [n.strip().split() for n in server_names],
        'locations':    locations,
        'proxy_passes': proxy_passes,
        'ssl_certs':    ssl_certs,
    }


tf_sample = """
resource "aws_s3_bucket" "main" {
  bucket = "my-app-storage"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  db_password = "hardcoded_password_123"
}

resource "aws_rds_instance" "db" {
  identifier = "my-database"
}
"""

print(f"\n   Terraform analysis:")
tf_analysis = analyze_terraform(tf_sample)
print(f"   Resources ({tf_analysis['resource_count']}):")
for rtype, rname in tf_analysis['resources']:
    print(f"     {rtype}.{rname}")
if tf_analysis['sensitive_values_found']:
    print(f"   WARNING: {tf_analysis['sensitive_values_found']} hardcoded sensitive value(s) found!")


nginx_sample = """
server {
    listen 443 ssl;
    server_name example.com www.example.com;
    ssl_certificate /etc/ssl/example.com.crt;
    ssl_certificate_key /etc/ssl/example.com.key;

    location / {
        proxy_pass http://backend:8080;
    }

    location = /health {
        return 200 "OK";
    }

    location ~ /api/v[0-9]+/ {
        proxy_pass http://api-backend:3000;
    }
}
"""

print(f"\n   Nginx config analysis:")
nginx_analysis = analyze_nginx_config(nginx_sample)
for key, val in nginx_analysis.items():
    print(f"   {key}: {val}")
```

---

## 94.3 Monitoring & Alerting Patterns

```python
import re
from typing import Dict, List, Optional

print("\nMonitoring & Alerting Patterns:")
print("=" * 60)

# Prometheus metric format parser
PROM_METRIC = re.compile(
    r'^(?P<name>[a-zA-Z_:][a-zA-Z0-9_:]*)'
    r'(?:\{(?P<labels>[^}]*)\})?'
    r'\s+(?P<value>[-+]?(?:inf|nan|\d+(?:\.\d+)?(?:[eE][+-]?\d+)?))'
    r'(?:\s+(?P<timestamp>\d+))?$'
)
PROM_LABEL = re.compile(r'(?P<key>[a-zA-Z_][a-zA-Z0-9_]*)="(?P<value>[^"]*)"')

# AWS ARN parser
AWS_ARN = re.compile(
    r'arn:(?P<partition>aws(?:-cn|-us-gov)?):(?P<service>[a-z0-9\-]+):'
    r'(?P<region>[a-z0-9\-]*):(?P<account>\d{12})?:'
    r'(?P<resource>[a-zA-Z0-9/:\-_.*]+)'
)


def parse_prometheus_line(line: str) -> Optional[Dict]:
    m = PROM_METRIC.match(line.strip())
    if not m:
        return None

    result = {
        'name':   m.group('name'),
        'value':  float(m.group('value')),
        'labels': {},
    }

    if m.group('labels'):
        for lm in PROM_LABEL.finditer(m.group('labels')):
            result['labels'][lm.group('key')] = lm.group('value')

    if m.group('timestamp'):
        result['timestamp'] = int(m.group('timestamp'))

    return result


prom_lines = [
    'http_requests_total{method="GET",status="200"} 1234',
    'http_requests_total{method="POST",status="500"} 42',
    'process_cpu_seconds_total 0.05 1705312345000',
    'node_memory_MemFree_bytes 1.234e+09',
    '# COMMENT LINE (should not parse)',
]

print(f"\n   Prometheus metric parsing:")
for line in prom_lines:
    result = parse_prometheus_line(line)
    if result:
        labels_str = ', '.join(f"{k}={v!r}" for k, v in result['labels'].items())
        print(f"   {result['name']}{{{labels_str}}} = {result['value']}")
    else:
        print(f"   SKIP: {line!r}")


cloud_resources = [
    'arn:aws:s3:::my-bucket',
    'arn:aws:iam::123456789012:role/MyRole',
    'arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0',
]

print(f"\n   AWS ARN parsing:")
for arn in cloud_resources:
    m = AWS_ARN.match(arn)
    if m:
        print(f"   {arn!r}")
        print(f"     service={m.group('service')!r}, region={m.group('region')!r}, resource={m.group('resource')!r}")
```

---

## 94.4 สรุป Part 94

```
Regex in DevOps & Infrastructure:

1. CI/CD patterns:
   Conventional Commits: type(scope)!: description
   Types: feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert
   Breaking change: ! after scope OR BREAKING CHANGE footer
   Semver bump: feat=minor, fix/perf=patch, BREAKING=major

2. Infrastructure as Code:
   Terraform: resource "type" "name" { ... }
   Detect hardcoded secrets: password|secret|token\s*=\s*"..."
   Nginx: server_name, location modifier+path, proxy_pass
   Ansible: become: yes = privilege escalation

3. Kubernetes validation:
   Resource names: ^[a-z0-9]([a-z0-9\-]{0,61}[a-z0-9])?$
   Labels: max 63 chars, optional prefix/63-char name
   Namespaces: same as resource names, max 63 chars

4. Monitoring:
   Prometheus: name{labels} value [timestamp]
   Labels: key="value" pairs inside {}
   Numeric values: int, float, sci notation, inf, nan

5. Cloud resource patterns:
   AWS ARN: arn:partition:service:region:account:resource
   GCP: projects/id/type/name
   Azure: /subscriptions/uuid/resourceGroups/rg/providers/p/t/name
```

---

*[← Part 93: Regex Testing & QA](part-93-testing-qa.md) | [→ Part 95: Regex Performance Optimization](part-95-performance.md)*
