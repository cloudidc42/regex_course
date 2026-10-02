# Part 81: Cloud Security Patterns

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~100 นาที | **ข้อกำหนด:** Part 01-80

---

## 81.1 AWS Security Pattern Detection

```python
import re
from typing import Dict, List

print("Cloud Security Pattern Detection:")
print("=" * 60)

print("\n1. AWS Security Patterns:")

# AWS ARN (Amazon Resource Name)
AWS_ARN = re.compile(
    r'arn:(?P<partition>aws|aws-cn|aws-us-gov):'
    r'(?P<service>[a-zA-Z0-9\-]+):'
    r'(?P<region>[a-z0-9\-]*):'
    r'(?P<account>\d{12}|):'
    r'(?P<resource>[^\s"\' ]+)'
)

# CloudTrail event fields
CT_EVENT_NAME = re.compile(r'"eventName"\s*:\s*"(?P<name>[^"]+)"')
CT_SOURCE_IP  = re.compile(r'"sourceIPAddress"\s*:\s*"(?P<ip>[^"]+)"')
CT_ERROR_CODE = re.compile(r'"errorCode"\s*:\s*"(?P<code>[^"]+)"')

# High-risk CloudTrail events
HIGH_RISK_EVENTS = {
    'CreateUser', 'CreateAccessKey', 'AttachUserPolicy',
    'AttachRolePolicy', 'CreateLoginProfile', 'UpdateLoginProfile',
    'AddUserToGroup', 'PutUserPolicy',
    'PutBucketPolicy', 'PutBucketAcl', 'DeleteBucketPolicy',
    'GetSecretValue', 'GetParameter', 'GetParametersByPath',
    'AuthorizeSecurityGroupIngress', 'CreateVpc', 'ModifyInstanceAttribute',
    'DeleteTrail', 'StopLogging', 'DeleteFlowLogs', 'DisableKey',
}

# S3 public access patterns
S3_PUBLIC_ACCESS = re.compile(
    r'"Principal"\s*:\s*(?:"\*"|\{"AWS"\s*:\s*"\*"\})',
    re.IGNORECASE
)

# IAM over-permissive
IAM_ADMIN_WILDCARD    = re.compile(r'"Action"\s*:\s*(?:"[*]"\s*|\[\s*"[*]"\s*\])')
IAM_RESOURCE_WILDCARD = re.compile(r'"Resource"\s*:\s*(?:"[*]"\s*|\[\s*"[*]"\s*\])')


def analyze_cloudtrail_event(event_json: str) -> Dict:
    findings = []
    name_m = CT_EVENT_NAME.search(event_json)
    event_name = name_m.group('name') if name_m else 'Unknown'

    if event_name in HIGH_RISK_EVENTS:
        findings.append(f'HIGH_RISK_EVENT: {event_name}')

    ip_m = CT_SOURCE_IP.search(event_json)
    source_ip = ip_m.group('ip') if ip_m else 'Unknown'

    err_m = CT_ERROR_CODE.search(event_json)
    if err_m:
        findings.append(f'ERROR: {err_m.group("code")}')

    if S3_PUBLIC_ACCESS.search(event_json):
        findings.append('S3_PUBLIC_ACCESS: Principal=* in policy')

    if IAM_ADMIN_WILDCARD.search(event_json) and IAM_RESOURCE_WILDCARD.search(event_json):
        findings.append('IAM_ADMIN_GRANT: Action=* Resource=* (full admin)')

    return {
        'event':    event_name,
        'src_ip':   source_ip,
        'findings': findings,
        'risk':     'HIGH' if findings else 'LOW',
    }


cloudtrail_events = [
    '{"eventName":"CreateAccessKey","sourceIPAddress":"203.0.113.42","userIdentity":{"type":"Root"}}',
    '{"eventName":"GetSecretValue","sourceIPAddress":"10.0.0.1","errorCode":"AccessDenied"}',
    '{"eventName":"PutBucketPolicy","Principal":"*","Action":"s3:*","Resource":"*"}',
    '{"eventName":"DescribeInstances","sourceIPAddress":"10.0.0.5"}',
    '{"eventName":"DeleteTrail","sourceIPAddress":"203.0.113.99"}',
    '{"eventName":"AttachUserPolicy","Action":"*","Resource":"*"}',
]

print(f"\n   CloudTrail event analysis:")
for event in cloudtrail_events:
    result = analyze_cloudtrail_event(event)
    risk_mark = '\U0001f6a8' if result['risk'] == 'HIGH' else '✓'
    print(f"\n   {risk_mark} Event: {result['event']} from {result['src_ip']}")
    for f in result['findings']:
        print(f"     → {f}")
```

---

## 81.2 Docker Security Audit

```python
import re
from typing import List

print("\nDocker Security Audit:")
print("=" * 60)

# Docker security anti-patterns
DOCKER_PRIV       = re.compile(r'--privileged', re.IGNORECASE)
DOCKER_HOST_MOUNT = re.compile(r'-v\s+/(?:etc|var|proc|root|home|usr):[^:]', re.IGNORECASE)
DOCKER_HOST_PID   = re.compile(r'--pid=host', re.IGNORECASE)
DOCKER_NET_HOST   = re.compile(r'--network=host', re.IGNORECASE)
DOCKER_CAPS       = re.compile(r'--cap-add\s+(?:ALL|SYS_ADMIN|SYS_PTRACE|NET_ADMIN)', re.IGNORECASE)
DOCKER_ROOT       = re.compile(r'--user\s+(?:root|0)', re.IGNORECASE)
DOCKER_NO_SECCOMP = re.compile(r'--security-opt\s+seccomp=unconfined', re.IGNORECASE)


def audit_docker_run(cmd: str) -> List[str]:
    findings = []
    if DOCKER_PRIV.search(cmd):
        findings.append('CRITICAL: --privileged (bypasses all namespaces)')
    m = DOCKER_HOST_MOUNT.search(cmd)
    if m:
        findings.append(f'HIGH: sensitive host mount: {m.group(0)}')
    if DOCKER_HOST_PID.search(cmd):
        findings.append('HIGH: host PID namespace (sees all processes)')
    if DOCKER_NET_HOST.search(cmd):
        findings.append('MEDIUM: host network namespace')
    m = DOCKER_CAPS.search(cmd)
    if m:
        findings.append(f'HIGH: dangerous capability: {m.group(0)}')
    if DOCKER_NO_SECCOMP.search(cmd):
        findings.append('HIGH: seccomp disabled (all syscalls allowed)')
    return findings


docker_commands = [
    'docker run -it ubuntu bash',
    'docker run --privileged -v /:/host ubuntu bash',
    'docker run --pid=host --network=host alpine sh',
    'docker run -v /etc:/host/etc:ro nginx',
    'docker run --cap-add ALL ubuntu bash',
    'docker run -p 8080:80 -e APP_ENV=prod nginx',
    'docker run --cap-add SYS_ADMIN ubuntu bash',
    'docker run --security-opt seccomp=unconfined ubuntu bash',
]

print(f"\n   Docker run security audit:")
for cmd in docker_commands:
    findings = audit_docker_run(cmd)
    if findings:
        print(f"\n   ⚠ {cmd[:70]!r}")
        for f in findings:
            print(f"     → {f}")
    else:
        print(f"   ✓ {cmd[:70]!r}")
```

---

## 81.3 Cloud Metadata & SSRF Detection

```python
import re
from typing import List

print("\nCloud Metadata & SSRF Detection:")
print("=" * 60)

# Cloud metadata service endpoints
CLOUD_METADATA = re.compile(
    r'(?:169\.254\.169\.254|'
    r'metadata\.google\.internal|'
    r'169\.254\.170\.2|'
    r'100\.100\.100\.200)',
    re.IGNORECASE
)

# AWS metadata credential paths
AWS_META_CREDS = re.compile(
    r'latest/meta-data/iam/security-credentials',
    re.IGNORECASE
)

# GCP metadata service account path
GCP_META_SA = re.compile(
    r'computeMetadata/v1/instance/service-accounts',
    re.IGNORECASE
)

# SSRF via open redirect parameters
SSRF_PARAM = re.compile(
    r'(?:url|redirect|callback|next|goto|target|return_url|'
    r'returnUrl|redirect_to|forward|dest|destination)\s*=\s*'
    r'(?:https?://[^\s&]+|//[^\s&]+)',
    re.IGNORECASE
)

# Internal network access
INTERNAL_SERVICES = re.compile(
    r'(?:localhost|127\.0\.0\.1|0\.0\.0\.0|'
    r'0x7f000001|2130706433|'
    r'10\.\d+\.\d+\.\d+|'
    r'172\.(?:1[6-9]|2\d|3[01])\.\d+\.\d+|'
    r'192\.168\.\d+\.\d+)',
    re.IGNORECASE
)

# Dangerous protocols for SSRF
DANGEROUS_PROTO = re.compile(
    r'(?:file://|dict://|gopher://|ftp://|sftp://)',
    re.IGNORECASE
)


def detect_ssrf(value: str) -> List[str]:
    findings = []
    if CLOUD_METADATA.search(value):
        findings.append('CRITICAL: Cloud metadata service access (169.254.169.254)')
        if AWS_META_CREDS.search(value):
            findings.append('CRITICAL: AWS IAM credentials endpoint')
        if GCP_META_SA.search(value):
            findings.append('CRITICAL: GCP service account token endpoint')
    if INTERNAL_SERVICES.search(value):
        findings.append('HIGH: Internal/loopback address access')
    m = DANGEROUS_PROTO.search(value)
    if m:
        findings.append(f'HIGH: Dangerous protocol: {m.group(0)}')
    return findings


ssrf_payloads = [
    'http://169.254.169.254/latest/meta-data/iam/security-credentials/',
    'http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/',
    'http://localhost:8080/admin',
    'http://192.168.1.1/router-admin',
    'file:///etc/passwd',
    'dict://localhost:11211/stat',
    'http://0x7f000001/admin',
    'https://trusted.example.com/api',
]

print(f"\n   SSRF detection:")
for payload in ssrf_payloads:
    findings = detect_ssrf(payload)
    if findings:
        print(f"\n   \U0001f6a8 {payload[:70]!r}")
        for f in findings:
            print(f"     → {f}")
    else:
        print(f"   ✓ {payload[:70]!r}")
```

---

## 81.4 สรุป Part 81

```
Cloud Security Pattern Design:

1. AWS CloudTrail monitoring:
   High-risk: CreateAccessKey, DeleteTrail, GetSecretValue, StopLogging
   IAM misconfig: Action=*, Resource=* (full admin)
   S3 exposure: Principal=* in bucket policy
   Root usage: userIdentity.type = Root (always alert)

2. Docker security:
   --privileged: bypass all namespaces (kernel access)
   -v /:/host: read entire host filesystem
   --pid=host: see all host processes
   --cap-add ALL/SYS_ADMIN: dangerous capabilities
   --security-opt seccomp=unconfined: all syscalls

3. SSRF / Cloud metadata:
   169.254.169.254: AWS/Azure/GCP metadata
   metadata.google.internal: GCP metadata
   /latest/meta-data/iam/: AWS IAM credentials!
   computeMetadata/v1/instance/service-accounts/: GCP SA token!
   Defense: Block 169.254.x.x at egress, require Metadata-Flavor header

4. Dangerous protocols for SSRF:
   file:// -> read local files
   dict:// -> probe open ports (Memcached, Redis)
   gopher:// -> send raw TCP (Redis, SMTP, SSRF to internal)
   ftp:// -> exfiltrate data to FTP server

5. Defense strategies:
   AWS: Use IMDSv2 (token-based metadata), block IMDSv1
   GCP: Require 'Metadata-Flavor: Google' header
   K8s: RBAC least privilege, PodSecurityAdmission
   Docker: Use rootless containers, no-new-privileges flag
   Network: Block 169.254.0.0/16 outbound from app servers
```

---

*[← Part 80: Log Analysis & SIEM](part-80-siem-patterns.md) | [→ Part 82: API Security Patterns](part-82-api-security.md)*
