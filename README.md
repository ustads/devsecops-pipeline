# Automated DevSecOps Pipeline: Multi-Scanner Integration & DefectDojo Centralization

![DevSecOps](https://img.shields.io/badge/Security-DevSecOps-red.svg)
![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-orange.svg)
![DefectDojo](https://img.shields.io/badge/Vulnerability--Management-DefectDojo-blue.svg)

Repositori ini berisi cetak biru (*blueprint*) implementasi pipeline **DevSecOps** menggunakan **Jenkins Declarative Pipeline**. Sistem ini menerapkan pendekatan keamanan *Shift-Left* dengan mengintegrasikan lima lapisan pemindaian keamanan (*multi-layered security scanning*) otomatis sebelum aplikasi dideploy, serta mengonsolidasikan seluruh metrik temuan ke dalam **DefectDojo** sebagai *Single Source of Truth*.

---

## 🏗️ Arsitektur & Alur Pipeline

Pipeline ini dirancang untuk berjalan secara berurutan (*sequential stages*) guna memastikan bahwa jika ada kerentanan fatal di tahap awal, pipa akan langsung berhenti (*fail-fast mechanism*):

```text
[Code Checkout] 
       │
       ▼
[1. Secret Scanning] ──────► TruffleHog (Detects hardcoded keys/tokens)
       │
       ▼
[2. IaC Scanning]     ──────► Checkov (Audits Terraform/Dockerfiles)
       │
       ▼
[3. SAST]             ──────► SonarQube (Static Application Security Testing)
       │
       ▼
[4. SCA]              ──────► OWASP Dependency-Check (Third-party libraries CVEs)
       │
       ▼
[5. FS Scanning]      ──────► Trivy (Deep filesystem vulnerability audit)
       │
       ▼
[6. Centralization]   ──────► DefectDojo API v2 (Ingestion, Deduplication & Tracking)
