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
```

---

## 🛠️ Komponen & Fungsi Perkakas

| Nama Perkakas | Kategori Keamanan | Fungsi Utama | Format Output |
|---|---|---|---|
| TruffleHog | Secret Scanning | Memindai riwayat git dan berkas dari kebocoran kredensial/token | CLI / Fail State |
| Checkov | IaC Security | Menganalisis miskonfigurasi pada berkas infrastruktur (Docker/Terraform) | JSON |
| SonarQube | SAST | Melakukan analisis statis kode sumber untuk celah keamanan logic (OWASP Top 10) | Web API -> JSON |
| Dependency-Check | SCA | Memeriksa kerentanan publik (CVE) pada pustaka (library) pihak ketiga | XML & HTML |
| Trivy | Vulnerability Scan | Pemindaian mendalam pada sistem berkas (filesystem) kontainer | JSON |
| DefectDojo | Vulnerability Management | Dasbor terpusat untuk mendeduplikasi dan melacak proses remediasi isu | Dashboard |

---

## 🚀 Langkah Demi Langkah Pengaturan (Step-by-Step Setup)

### Langkah 1: Persiapan Lingkungan Network (Docker Gateway)

Jika Jenkins dan server pemindai berjalan di lingkungan kontainer terpisah/lokal, pastikan komunikasi antar-kontainer stabil menggunakan IP Gateway Docker Network internal agar terhindar dari isu IP dinamis:

1. **Tentukan IP Gateway jaringan Anda** (Contoh: `172.18.0.1`)
2. **Pastikan port target terekspos** seperti SonarQube (9000) dan DefectDojo (8085) terekspos (forwarded) ke host

```bash
# Cek IP Gateway Docker Anda
docker network inspect bridge | grep Gateway

# Atau gunakan IP container tertentu
docker inspect -f '{{range .NetworkSettings.Networks}}{{.Gateway}}{{end}}' <container_name>
```

### Langkah 2: Konfigurasi Credentials di Jenkins

Sebelum menjalankan pipeline, daftarkan rahasia berikut pada menu **Manage Jenkins ➡️ Credentials**:

#### 2.1 SonarQube Token
- **Credential ID**: `sonarqube-token` (Secret Text)
- **Fungsi**: Token otentikasi dari akun SonarQube Anda
- **Cara mendapat**: SonarQube Dashboard → User Profile → Security → Generate Tokens

#### 2.2 NVD API Key
- **Credential ID**: `nvd-api-key` (Secret Text)
- **Fungsi**: API Key resmi dari NVD untuk mempercepat pembaharuan basis data OWASP Dependency-Check
- **Cara mendapat**: https://nvd.nist.gov/developers/request-an-api-key

#### 2.3 DefectDojo API Token
- **Credential ID**: `defectdojo-api-token` (Secret Text)
- **Fungsi**: API v2 Token yang digenerate dari profil pengguna DefectDojo Anda
- **Cara mendapat**: DefectDojo Dashboard → User Profile → API Token → Generate Token

**Langkah Konfigurasi di Jenkins:**
```
1. Navigate: Manage Jenkins → Manage Credentials
2. Click: Add Credentials
3. Kind: Secret text
4. Secret: Paste API token
5. ID: sonarqube-token (atau sesuai nama credential)
6. Click: Create
7. Repeat untuk nvd-api-key dan defectdojo-api-token
```

### Langkah 3: Konfigurasi Tools Global

Pastikan perkakas berikut sudah terpasang dan dikonfigurasi namanya pada menu **Manage Jenkins ➡️ Global Tool Configuration**:

#### 3.1 SonarQube Scanner Configuration
```
Name: SonarQubeScanner
Install automatically: ✓ Checked
Version: Latest (atau versi spesifik)
```

**Langkah konfigurasi:**
```
1. Navigate: Manage Jenkins → Global Tool Configuration
2. Scroll to: SonarQube Scanner
3. Click: Add SonarQube Scanner
4. Name: SonarQubeScanner (WAJIB sama dengan Jenkinsfile)
5. Install automatically: ✓ Checked
6. Version: Select latest version
7. Click: Save
```

#### 3.2 OWASP Dependency-Check Configuration
```
Name: DP-Check
Install automatically: ✓ Checked
Version: Latest (atau versi spesifik)
```

**Langkah konfigurasi:**
```
1. Navigate: Manage Jenkins → Global Tool Configuration
2. Scroll to: Dependency-Check
3. Click: Add Dependency-Check
4. Name: DP-Check (WAJIB sama dengan Jenkinsfile)
5. Install automatically: ✓ Checked
6. Version: Select latest version
7. Click: Save
```

#### 3.3 SonarQube Server Configuration
```
Name: SonarQube
Server URL: http://<IP_GATEWAY_DOCKER>:9000
Credential: sonarqube-token
```

**Langkah konfigurasi:**
```
1. Navigate: Manage Jenkins → Configure System
2. Scroll to: SonarQube Servers
3. Click: Add SonarQube
4. Name: SonarQube
5. Server URL: http://172.18.0.1:9000 (sesuaikan IP & port)
6. Server authentication token: Select sonarqube-token
7. Click: Save
```

---

## 📄 Cetak Biru Jenkinsfile

Salin berkas konfigurasi pipeline utama dan letakkan dengan nama `Jenkinsfile` di root repositori proyek Anda. Lihat file `pipelines/Jenkinsfile` untuk detail lengkap.

### Environment Variables yang Perlu Dikonfigurasi:

```groovy
environment {
    SONARQUBE_URL   = 'http://172.18.0.1:9000'  // Ganti dengan IP & port Anda
    TRIVY_VERSION   = '0.52.2'                   // Version Trivy (opsional update)
    DEFECTDOJO_URL  = 'http://172.18.0.1:8085'  // Ganti dengan IP & port Anda
    ENGAGEMENT_ID   = '1'                        // Ganti dengan Engagement ID DefectDojo Anda
}
```

### Project Key SonarQube:

Ganti `<YOUR_PROJECT_KEY>` di Jenkinsfile dengan project key Anda:
```groovy
-Dsonar.projectKey=my-awesome-project
```

---

## 💡 Fitur Unggulan Sistem

### ✅ Auto-Mitigation Terotomatisasi
Menggunakan parameter `close_old_findings=true` pada DefectDojo API, sehingga temuan lama yang sudah diperbaiki oleh developer pada commit terbaru akan otomatis ditutup secara sistematis.

**Implementasi:**
```groovy
// Contoh di Jenkinsfile
sh "curl -X POST '${DEFECTDOJO_URL}/api/v2/import-scan/' \
    -H 'Authorization: Token \${DOJO_TOKEN}' \
    -F 'scan_date=\$(date +%Y-%m-%d)' \
    -F 'scan_type=SonarQube Scan' \
    -F 'engagement=${ENGAGEMENT_ID}' \
    -F 'file=@sonarqube-report.json' \
    -F 'close_old_findings=true'"
```

**Benefit:**
- 🔄 Otomatis menutup findings yang sudah di-remediate
- 📊 Mengurangi false positives di dashboard
- ⚡ Efisiensi tracking status vulnerability

---

### ✅ Keamanan Enkapsulasi Shell
Menggunakan taktik backslash escaping (`\$TOKEN`) pada Jenkins script untuk menghindari celah keamanan Groovy string interpolation sekaligus menjaga integrasi variabel lokal tetap terbaca sempurna.

**Implementasi:**
```groovy
// ✅ AMAN - Menggunakan backslash escaping
withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
    sh "curl -s -u \$SONAR_TOKEN: \"${SONARQUBE_URL}/api/issues/search\""
}

// ❌ TIDAK AMAN - String interpolation langsung
sh "curl -s -u ${SONAR_TOKEN}: \"${SONARQUBE_URL}/api/issues/search\""
```

**Benefit:**
- 🔐 Mencegah credential leakage di logs
- 🛡️ Menghindari Groovy injection attacks
- 📝 Variabel tetap terbaca dan functional
- 🔒 Secret tokens tidak terekspos saat script debugging

---

### ✅ Penyimpanan Laporan Fisik
Memaksa perkakas berbasis CLI untuk mengekspor dokumen data fisik mentah (JSON/XML) agar validasi artefak kepatuhan (compliance audit) dapat dilakukan kapan saja.

**Implementasi:**
```groovy
// TruffleHog
sh "trufflehog filesystem ${WORKSPACE} --fail"

// Checkov
sh "docker run --rm ... bridgecrew/checkov -d ${WORKSPACE} --output json > checkov-report.json"

// SonarQube
sh "curl -s ... > sonarqube-report.json"

// Dependency-Check
dependencyCheck(
    odcInstallation: 'DP-Check', 
    additionalArguments: "--format HTML --format XML --out ."
)

// Trivy
sh "docker run --rm aquasec/trivy:${TRIVY_VERSION} fs --format json -o trivy-report.json"
```

**Benefit:**
- 📄 Laporan tersimpan untuk audit trail
- ✅ Compliance documentation yang lengkap
- 🔍 Dapat diverifikasi secara manual atau otomatis
- 💾 Archival untuk historical tracking
- 📋 Multi-format reporting (JSON, XML, HTML)

---

### ✅ Multi-Scanner Integration
Terintegrasi dengan 5 lapisan pemindaian keamanan berbeda dalam satu pipeline untuk coverage keamanan maksimal.

**Coverage:**
- 🔐 **Secret Scanning** → TruffleHog (Hardcoded credentials)
- 🏗️ **IaC Security** → Checkov (Infrastructure misconfigurations)
- 🔎 **SAST** → SonarQube (Code vulnerabilities)
- 📦 **SCA** → Dependency-Check (Third-party CVEs)
- 🖥️ **Filesystem Scanning** → Trivy (Container vulnerabilities)

**Benefit:**
- 🎯 Comprehensive security validation
- 🚀 Shift-left security approach
- 🔁 Automated fail-fast mechanism
- 📊 Centralized reporting di DefectDojo

---

### ✅ Fail-Fast Mechanism
Pipeline akan berhenti otomatis jika ada kritical vulnerability di tahap awal, mencegah deployment kode tidak aman.

**Implementasi:**
```groovy
stage('2. Secret Scanning (TruffleHog)') {
    steps {
        sh "trufflehog filesystem ${WORKSPACE} --fail"  // --fail akan exit code 1 jika ada secret
    }
}
```

**Benefit:**
- ⛔ Immediate halt pada security issues
- 💰 Menghemat resources CI/CD
- 🚫 Prevent unsafe code deployment
- 📞 Fast notification ke development team
- ⚡ Reduce remediation time

---

## 📁 Struktur Repositori

```
devsecops-pipeline/
├── .github/
│   └── workflows/          # (Opsional jika nanti ekspansi ke GitHub Actions)
├── pipelines/
│   └── Jenkinsfile         # Berkas Jenkinsfile utama yang sudah teroptimasi
├── scripts/
│   └── (Skrip otomasi tambahan jika ada)
└── README.md               # Dokumentasi utama (file ini)
```

---

## 🔧 Troubleshooting

### Issue: TruffleHog tidak terinstall
**Solusi**: Pipeline akan otomatis menginstall TruffleHog saat pertama kali dijalankan

### Issue: Koneksi ke SonarQube gagal
**Solusi**: Pastikan IP Gateway dan port sudah benar, serta SonarQube server sedang running

### Issue: DefectDojo tidak menerima report
**Solusi**: 
- Pastikan API token benar
- Pastikan Engagement ID valid
- Cek response dari curl command di logs Jenkins

### Issue: Trivy scan timeout
**Solusi**: Naikkan timeout atau gunakan volume mount yang lebih optimal

---

## 📝 Lisensi

Repository ini dapat digunakan sebagai template blueprint untuk implementasi DevSecOps di organisasi Anda.

---

## 📞 Support & Kontribusi

Untuk kontribusi atau pertanyaan, silakan buat issue atau pull request di repository ini.
