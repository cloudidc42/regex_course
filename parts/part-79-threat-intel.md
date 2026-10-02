# Part 79: Threat Intelligence Patterns

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~95 นาที | **ข้อกำหนด:** Part 01-78

---

## 79.1 IOC (Indicator of Compromise) Extraction

```python
import re
from typing import Dict, List, Set

print("Threat Intelligence Pattern Extraction:")
print("=" * 60)

print("\n1. IOC extraction patterns:")

# IP addresses (v4 strict)
IOC_IPV4 = re.compile(
    r'\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b'
)

# IPv6
IOC_IPV6 = re.compile(
    r'\b(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}\b'
    r'|(?:[0-9a-fA-F]{1,4}:){1,7}:'
    r'|::(?:[0-9a-fA-F]{1,4}:){0,6}[0-9a-fA-F]{1,4}'
)

# Domain names (including subdomains)
IOC_DOMAIN = re.compile(
    r'\b(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}\b'
)

# URLs (http/https/ftp)
IOC_URL = re.compile(
    r'https?://[^\s<>"\' ]{4,}'
    r'|ftp://[^\s<>"\' ]{4,}'
)

# File hashes
IOC_MD5    = re.compile(r'\b[0-9a-fA-F]{32}\b')
IOC_SHA1   = re.compile(r'\b[0-9a-fA-F]{40}\b')
IOC_SHA256 = re.compile(r'\b[0-9a-fA-F]{64}\b')

# Email addresses
IOC_EMAIL = re.compile(
    r'\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b'
)

# CVE identifiers
IOC_CVE = re.compile(r'\bCVE-\d{4}-\d{4,7}\b', re.IGNORECASE)

# MITRE ATT&CK technique IDs
IOC_MITRE = re.compile(r'\bT\d{4}(?:\.\d{3})?\b')

# Registry keys (Windows)
IOC_REGKEY = re.compile(
    r'(?:HKEY_LOCAL_MACHINE|HKEY_CURRENT_USER|HKLM|HKCU|HKU|HKCR|HKCC)'
    r'(?:\\[A-Za-z0-9_\-\. ]+)+',
    re.IGNORECASE
)

# File paths
IOC_WINPATH  = re.compile(r'[A-Za-z]:\\(?:[^\\\/:*?"<>|\r\n]+\\)*[^\\\/:*?"<>|\r\n]+')
IOC_UNIXPATH = re.compile(r'(?:/[a-zA-Z0-9_.\-]+){2,}')


def extract_iocs(text: str) -> Dict[str, Set[str]]:
    return {
        'ipv4':    set(IOC_IPV4.findall(text)),
        'ipv6':    set(IOC_IPV6.findall(text)),
        'domains': set(IOC_DOMAIN.findall(text)),
        'urls':    set(IOC_URL.findall(text)),
        'md5':     set(IOC_MD5.findall(text)),
        'sha1':    set(IOC_SHA1.findall(text)),
        'sha256':  set(IOC_SHA256.findall(text)),
        'emails':  set(IOC_EMAIL.findall(text)),
        'cves':    set(IOC_CVE.findall(text)),
        'mitre':   set(IOC_MITRE.findall(text)),
        'regkeys': set(IOC_REGKEY.findall(text)),
    }


sample_threat_report = """
Incident Report 2024-02-15:
Malware C2 communication detected from 192.168.10.55 to 185.220.101.42
Secondary C2: evil-c2.example.net (resolved to 10.0.0.100)
Payload download from https://malware-host.example.com/loader.exe

File indicators:
  loader.exe MD5: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6
  payload.dll SHA1: deadbeefcafebabe1234567890abcdef12345678
  dropper.ps1 SHA256: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef

Attacker email: attacker@evil-domain.example.com
Vulnerability exploited: CVE-2024-1234 (CVSS 9.8)
MITRE ATT&CK: T1059.001 (PowerShell), T1071.001 (HTTP C2)

Registry persistence:
HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run\\MalwareService

Files created:
C:\\Users\\Admin\\AppData\\Roaming\\evil\\malware.exe
/tmp/.hidden/backdoor.sh
"""

iocs = extract_iocs(sample_threat_report)
print(f"\n   IOC extraction from threat report:")
for ioc_type, values in iocs.items():
    if values:
        print(f"\n   [{ioc_type.upper()}]")
        for v in sorted(values):
            print(f"   \u2022 {v[:80]}")
```

---

## 79.2 STIX/TAXII Pattern Matching

```python
import re
from typing import Dict, List

print("\nSTIX Pattern Language Support:")
print("=" * 60)

# STIX 2.x pattern syntax (simplified detector)
# [type:property = 'value']

STIX_PATTERN = re.compile(
    r'\[(?P<type>[a-z-]+):(?P<prop>[a-z0-9._\[\]\'"\-]+)\s*'
    r'(?P<op>=|!=|LIKE|MATCHES|ISSUBSET|ISSUPERSET|IN|>|<|>=|<=)\s*'
    r"(?P<value>'[^']*'|\"[^\"]*\"|[\d.]+|\[[^\]]+\])"
    r'\]'
)

# Supported STIX observable types
STIX_TYPES = {
    'ipv4-addr', 'ipv6-addr', 'domain-name', 'url', 'file',
    'email-addr', 'email-message', 'network-traffic', 'process',
    'windows-registry-key', 'user-account', 'autonomous-system',
}


def parse_stix_patterns(stix_rule: str) -> List[Dict]:
    patterns = []
    for m in STIX_PATTERN.finditer(stix_rule):
        obj_type = m.group('type')
        patterns.append({
            'type':     obj_type,
            'property': m.group('prop'),
            'operator': m.group('op'),
            'value':    m.group('value').strip("'\"" ),
            'valid':    obj_type in STIX_TYPES,
        })
    return patterns


stix_rules = [
    "[ipv4-addr:value = '192.168.1.1']",
    "[domain-name:value = 'evil.example.com']",
    "[file:hashes.MD5 = 'a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6']",
    "[network-traffic:dst_port = 4444]",
    "[process:command_line MATCHES '.*powershell.*-enc.*']",
    "[email-addr:value = 'attacker@evil.example.com']",
    "[url:value LIKE 'https://malware%']",
    "[windows-registry-key:key = 'HKLM\\\\SOFTWARE\\\\malware']",
    "[invalid-type:value = 'test']",
]

print(f"\n   STIX 2.x pattern parsing:")
for rule in stix_rules:
    patterns = parse_stix_patterns(rule)
    if patterns:
        for p in patterns:
            valid_mark = '\u2713' if p['valid'] else '\u26a0 invalid type'
            print(f"   {valid_mark} {p['type']}:{p['property']} {p['operator']} {p['value']!r}")
    else:
        print(f"   \u2717 Could not parse: {rule!r}")
```

---

## 79.3 Malware Family Signature Patterns

```python
import re
from typing import Dict, List

print("\nMalware Family Signature Patterns:")
print("=" * 60)

MALWARE_SIGS = {
    'mimikatz_cmdline': [
        re.compile(r'sekurlsa::logonpasswords', re.IGNORECASE),
        re.compile(r'lsadump::(?:sam|dcsync|secrets)', re.IGNORECASE),
        re.compile(r'privilege::debug', re.IGNORECASE),
        re.compile(r'kerberos::(?:ptt|golden|silver)', re.IGNORECASE),
    ],
    'powershell_cradle': [
        re.compile(r'iex\s*\(|invoke-expression\s*\(', re.IGNORECASE),
        re.compile(r'(?:webclient|webrequest).*(?:downloadstring|downloadfile)', re.IGNORECASE),
        re.compile(r'\[convert\]::frombase64string\s*\(', re.IGNORECASE),
        re.compile(r'-encodedcommand\s+[A-Za-z0-9+/=]{20,}', re.IGNORECASE),
    ],
    'sql_injection_tool': [
        re.compile(r'sqlmap/[\d.]+', re.IGNORECASE),
        re.compile(r"WAITFOR\s+DELAY\s+'0:0:\d+'", re.IGNORECASE),
        re.compile(r'AND\s+\d+=\d+\s+AND\s+SUBSTRING\(', re.IGNORECASE),
    ],
    'webshell': [
        re.compile(r'(?:eval|assert|passthru|shell_exec|system)\s*\(\s*\$(?:_POST|_GET|_REQUEST|_COOKIE)', re.IGNORECASE),
        re.compile(r'base64_decode\s*\(\s*\$(?:_POST|_GET|_REQUEST|_COOKIE)', re.IGNORECASE),
        re.compile(r'@\s*(?:eval|assert)\s*\(', re.IGNORECASE),
    ],
}


def classify_threat(text: str) -> Dict[str, List[str]]:
    matches = {}
    for family, patterns in MALWARE_SIGS.items():
        hits = []
        for pattern in patterns:
            m = pattern.search(text)
            if m:
                hits.append(m.group(0)[:60])
        if hits:
            matches[family] = hits
    return matches


threat_samples = [
    "sekurlsa::logonpasswords\nlsadump::sam",
    "powershell -EncodedCommand JABzAD0AbgBlAHcALQBvAGIAagBlAGMAdAAgAFMAeQBzAHQAZQBtAC4ATgBlAHQALg==",
    "<?php @eval($_POST['cmd']); ?>",
    "AND 1=1 AND SUBSTRING(username,1,1)='a'",
    "Normal web server log entry",
]

print(f"\n   Malware family classification:")
for sample in threat_samples:
    matches = classify_threat(sample)
    short = sample[:60].replace('\n', '|')
    if matches:
        print(f"\n   THREAT DETECTED in: {short!r}")
        for family, hits in matches.items():
            print(f"   Family: {family}")
            for h in hits:
                print(f"   Match: {h!r}")
    else:
        print(f"   \u2713 CLEAN: {short!r}")
```

---

## 79.4 Threat Feed Integration

```python
import re
from typing import Dict, List, Optional

print("\nThreat Feed Integration Patterns:")
print("=" * 60)

# AlienVault OTX indicator format
OTX_INDICATOR = re.compile(
    r'"indicator"\s*:\s*"(?P<value>[^"]+)".*?'
    r'"type"\s*:\s*"(?P<type>[^"]+)"',
    re.DOTALL
)

# Abuse.ch URLHaus format (CSV)
URLHAUS_LINE = re.compile(
    r'^(?P<id>\d+),(?P<dateadded>[^,]+),(?P<url>"[^"]+"|[^,]+),'
    r'(?P<url_status>[^,]+),(?P<threat>[^,]+),(?P<tags>[^,\r\n]*)',
    re.MULTILINE
)

# Threat confidence scoring
THREAT_CONFIDENCE = re.compile(r'"confidence"\s*:\s*(\d+)')
THREAT_SEVERITY   = re.compile(r'"severity"\s*:\s*"(critical|high|medium|low|info)"', re.IGNORECASE)


def parse_threat_feed_line(line: str) -> Optional[Dict]:
    m = OTX_INDICATOR.search(line)
    if m:
        return {'source': 'otx', 'value': m.group('value'), 'type': m.group('type')}
    m = URLHAUS_LINE.match(line)
    if m:
        return {
            'source': 'urlhaus',
            'id':     m.group('id'),
            'url':    m.group('url').strip('"'),
            'status': m.group('url_status'),
            'threat': m.group('threat'),
        }
    return None


sample_feed_lines = [
    '{"indicator": "185.220.101.42", "type": "IPv4", "confidence": 95}',
    '{"indicator": "evil-c2.example.net", "type": "domain", "confidence": 80}',
    '1234,2024-02-15 10:00:00,"https://malware-host.example.com/loader.exe",online,malware_download,exe botnet',
    '5678,2024-02-15 11:00:00,"https://phishing.example.com/login",online,phishing,credential',
    'normal log line without threats',
]

print(f"\n   Threat feed parsing:")
for line in sample_feed_lines:
    result = parse_threat_feed_line(line)
    if result:
        print(f"   \u2713 {result}")
    else:
        conf_m = THREAT_CONFIDENCE.search(line)
        conf = f" (confidence={conf_m.group(1)})" if conf_m else ""
        print(f"   - Unparsed{conf}: {line[:60]!r}")
```

---

## 79.5 สรุป Part 79

```
Threat Intelligence Pattern Design:

1. IOC extraction (Indicator of Compromise):
   IPv4/IPv6: strict octets, no 0.0.0.0 or 255.255.255.255
   Hashes: MD5=32hex, SHA1=40hex, SHA256=64hex
   CVE: CVE-YYYY-NNNNN (4-7 digit NVD identifier)
   MITRE ATT&CK: T1234 or T1234.001 (sub-technique)
   Registry: HKLM/HKCU/HKU prefix + backslash path

2. STIX 2.x pattern language:
   [type:property operator 'value']
   Types: ipv4-addr, domain-name, file, network-traffic, process
   Operators: =, !=, LIKE, MATCHES, IN, ISSUBSET, ISSUPERSET
   Properties: file:hashes.MD5, network-traffic:dst_port

3. Malware signatures:
   PowerShell cradle: IEX(WebClient.DownloadString), -EncodedCommand
   Mimikatz: sekurlsa::logonpasswords, lsadump::sam
   Webshells: eval($_POST), base64_decode($_GET)

4. Threat feed formats:
   OTX: {"indicator": "...", "type": "IPv4", "confidence": N}
   URLHaus: id,date,url,status,threat,tags (CSV)
   MISP: {"value": "...", "type": "ip-dst", "category": "Network"}

5. Confidence scoring:
   95-100: HIGH confidence, auto-block
   70-94:  MEDIUM confidence, alert + investigate
   40-69:  LOW confidence, log only
   < 40:   Informational, threat hunting only
```

---

*[\u2190 Part 78: Advanced Evasion Analysis](part-78-advanced-evasion.md) | [\u2192 Part 80: Log Analysis & SIEM Patterns](part-80-siem-patterns.md)*
