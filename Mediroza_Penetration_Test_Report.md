# PENETRATION TESTING REPORT

## Mediroza General Hospital

| | |
|---|---|
| **Client** | Mediroza General Hospital |
| **Target** | https://medirozahospital.com |
| **Engagement Type** | Black-box Penetration Test & Vulnerability Assessment |
| **Duration** | 5 Days |
| **Author** | Shiv Kumar Das |
| **Programme** | NetworkWalks — Batch B083, Week 4 |
| **Report Date** | 3 October 2026 |
| **Classification** | CONFIDENTIAL — Authorised Personnel Only |

> **Authorisation:** This assessment was conducted against a target explicitly authorised for
> security testing by NetworkWalks for educational purposes. All techniques described herein
> were performed with written permission and must never be applied to any system without
> equivalent authorisation.

---

## 01 — Executive Summary

Mediroza General Hospital commissioned a black-box penetration test of its public web
infrastructure at `https://medirozahospital.com`. The engagement was conducted from the
perspective of an external attacker with no prior knowledge of the environment, and was
scoped to the hospital's web application and directly exposed server resources.

The assessment was **successful in achieving complete compromise of the confidentiality
objective**. In the course of a single working session, the tester:

1. **Bypassed the patient portal's authentication entirely** using a SQL injection flaw,
   gaining access to a restricted area of the site without any valid credentials.
2. **Retrieved three confidential patient pathology reports** (S. Dlamini, P. Reddy and
   E. Thompson), each of which was password-protected.
3. **Defeated the encryption on all three reports** within seconds, recovering the full
   medical contents of each.
4. **Identified a further critical exposure on the server** — an unauthenticated database
   backup — by analysing the hidden metadata of one of the recovered reports. This backup
   contained the **salaries of all 30 hospital employees** and the **complete shareholder
   register** of the hospital.

### Overall Risk: CRITICAL

The combination of a trivially exploitable authentication bypass and an exposed database
backup means the hospital's most sensitive data — patient medical records, employee payroll
data, national identity numbers and corporate ownership structure — is effectively public.
An attacker with no specialist skill could reproduce the entire attack chain using freely
available tools in under an hour.

| # | Finding | Severity |
|---|---|---|
| V-01 | SQL Injection — Authentication Bypass in Patient Portal | **Critical** |
| V-02 | Exposed Database Backup Containing PII, Payroll and Shareholder Data | **Critical** |
| V-03 | Weak / Predictable Passwords Protecting Encrypted Patient Reports | **High** |
| V-04 | Directory Listing Enabled on Multiple Server Paths | **High** |
| V-05 | Sensitive Information Disclosure via PDF Metadata | **Medium** |

---

## 02 — Scope and Methodology

### 2.1 Scope

| Item | Detail |
|---|---|
| In scope | `https://medirozahospital.com` and all web-accessible resources beneath it |
| Out of scope | Any system, domain or IP address not owned by the client |
| Prohibited | Social engineering, denial-of-service, and any destructive testing |
| Testing window | 5 days (all objectives achieved within the first day) |

### 2.2 Methodology

The engagement followed a standard black-box penetration testing lifecycle, structured to
satisfy the four milestones defined in the assignment:

```
  MILESTONE 1              MILESTONE 2              MILESTONE 3
  Initial Access     -->   Data Extraction    -->   Attack (Cracking)
  Recon, identify          Crack encryption          Analyse metadata,
  entry point, gain         on the 3 PDFs,            trace to further
  unauthorised access       recover contents          server exposure
        |                                                        |
        +------------------>  MILESTONE 4  <---------------------+
                             Professional Report
```

Each phase was executed manually and deliberately, with evidence captured at every step to
support the final report.

### 2.3 Tools Used

| Phase | Tool | Purpose |
|---|---|---|
| Reconnaissance | `curl` | HTTP requests, header analysis, robots.txt and directory enumeration |
| Reconnaissance | Browser / raw HTML review | Login form and input field analysis |
| Exploitation | `curl` (crafted POST) | Manual SQL injection authentication bypass |
| Analysis | `qpdf --show-encryption` | Identify PDF encryption scheme and parameters |
| Analysis | `pdf2john` | Extract password hash from encrypted PDFs |
| Cracking | `pdfcrack` | Dictionary attack against the extracted hashes |
| Cracking | `rockyou.txt` | Wordlist of ~14.3 million real-world leaked passwords |
| Decryption | `qpdf --decrypt` | Remove encryption using recovered passwords |
| Forensics | `exiftool` | Extract hidden document metadata |
| Extraction | `sed`, `grep` | Parse the recovered SQL backup into readable records |

### 2.4 Limitations

- The engagement was time-boxed; deeper post-exploitation (e.g. attempting to reach the
  database server itself) was not performed as it fell outside the agreed scope.
- No automated vulnerability scanner was used — all findings were confirmed manually and
  reproducibly, which reduces false positives.
- `john --format=PDF` failed to load the extracted hashes on the tester's build (see
  Section 3.2.4); `pdfcrack` was used as an equivalent and verified alternative. This is a
  tooling limitation, not a finding against the client.

---

## 03 — Findings and Proof of Exploitation

### 3.1 Milestone 1 — Initial Access

#### 3.1.1 Reconnaissance

The first step in any black-box test is to map the attack surface. The hospital's
`robots.txt` file — intended to instruct search engines which paths to ignore — disclosed
three directories that immediately became priority targets:

![robots.txt](screenshots/01-recon-robots.png)

```
Disallow: /patient/
Disallow: /staff/
Disallow: /old/
```

Directory enumeration of `/old/` and `/patient/` revealed that the server had **directory
listing enabled** on both paths, exposing the complete file inventory:

![Directory listing](screenshots/02-directory-listing.png)

Two observations were immediately significant:

- `/old/mediroza_db_backup_2019.sql` — a database backup file publicly downloadable.
- `/patient/` contained `login.php`, `portal.php`, `download.php` and a `reports/` folder,
  indicating an authenticated patient portal with document download functionality.

#### 3.1.2 Authentication Mechanism Analysis

The patient portal login form was inspected to understand how credentials were submitted:

![Login page](screenshots/03-login-page.png)

```html
<form method="POST" action="login.php">
  <input type="text"     name="username">
  <input type="password" name="password">
</form>
```

The form performs a simple POST of `username` and `password` to `login.php`. This is the
classic profile of an application that builds a database query by directly concatenating
user input — the precondition for SQL injection.

#### 3.1.3 Exploitation — SQL Injection Authentication Bypass

A standard authentication-bypass payload was submitted, using a deliberately **invalid**
password to prove that the password check was being discarded entirely:

```bash
curl -s -i -X POST https://medirozahospital.com/patient/login.php \
     -d "username=admin'--&password=x"
```

**Payload analysis:** the `'` character closes the string literal in the SQL query, and the
`--` sequence comments out the remainder of the statement — including the password
comparison. The query effectively becomes *"return the user named admin, ignoring
everything after"*.

![SQL injection auth bypass](screenshots/04-sqli-auth-bypass.png)

The server responded with:

```
HTTP/2 302
location: portal.php
set-cookie: PHPSESSID=cspab80ndg3bms3nf89mib0thj
```

A `302` redirect to `portal.php` together with a freshly issued session cookie is
conclusive proof that authentication was bypassed. **No valid credentials were used at any
point.**

#### 3.1.4 Accessing the Restricted Area

Using the session cookie, the restricted portal was accessed and three confidential
patient reports were enumerated:

![Patient portal](screenshots/05-portal-reports.png)

| Report | Patient | Lab Reference | Date |
|---|---|---|---|
| `download.php?id=1` | Pathology Report — S. Dlamini | LR-2024-1187 | 2024-11-04 |
| `download.php?id=2` | Pathology Report — P. Reddy | LR-2024-1192 | 2024-11-05 |
| `download.php?id=3` | Pathology Report — E. Thompson | LR-2024-1205 | 2024-11-06 |

All three files were retrieved:

![Downloaded PDFs](screenshots/06-downloaded-pdfs.png)

> **MILESTONE 1 DELIVERABLE ACHIEVED** — unauthorised access to a restricted area of the
> site, with proof of access and all three confidential patient PDF files retrieved.

### 3.2 Milestone 2 — Data Extraction (Cracking the Encryption)

The portal warned that *"Your reports are password protected"*. Each file was therefore
analysed to determine its protection scheme before selecting an attack.

#### 3.2.1 Encryption Analysis

```bash
qpdf --show-encryption report1.pdf
```

![Encryption analysis](screenshots/07-pdf-encryption-info.png)

All three files shared an identical protection profile:

| Parameter | Value | Meaning |
|---|---|---|
| Security Handler | `Standard` | The classic PDF password scheme |
| `R` (Revision) | `3` | Algorithm 3.2 / 3.5 — MD5 + RC4 |
| `P` (Permissions) | `-4` | All operations permitted |
| Key length | `128` bits | RC4 with a 128-bit key |

**Analytical conclusion:** because the *method* was identical across all three files, the
only variable was **password strength**. This directly informed the choice of a dictionary
attack rather than a computational brute-force, and validated the assignment's caution that
*"a single approach"* should not be assumed to suit every file.

#### 3.2.2 Hash Extraction

The PDF password verifier was extracted into a crackable hash using `pdf2john`:

![pdf2john hash extraction](screenshots/08-pdf2john-hashes.png)

```
report1.pdf:$pdf$2*3*128*294967292*1*32*<FileID>*32*<U>*32*<O>
```

Decoding this structure: `$pdf$` identifies a PDF hash, `2` the format version, `3` the
revision, `128` the key length, followed by the permission value, metadata flag, file ID,
and the `U`/`O` verification strings.

#### 3.2.3 Tooling Obstacle and Root-Cause Analysis

The initial cracking attempt failed:

![John the Ripper load error](screenshots/09-john-error-load.png)

```
$ john --wordlist=/usr/share/wordlists/rockyou.txt hash1.txt
Using default input encoding: UTF-8
No password hashes loaded (see FAQ)
```

**Investigation.** The `/P` permissions field had been serialised by `pdf2john` as
`294967292` rather than the correct signed value `-4`. This is a documented upstream defect
in John the Ripper (openwall/john issue #4739, *"pdf2john — Unsupported PDF files?"*), in
which the project maintainer confirms the workaround of *"manually replacing it in
output.hash"*.

The value was corrected and re-tested:

```bash
sed -i 's/294967292/-4/' hash1.txt
```

![Corrected hash, still failing](screenshots/09b-john-still-failing.png)

The corrected hash was independently validated against both current and legacy versions of
John's `pdf_valid()` parser and confirmed structurally sound — establishing that the
obstacle lay in the tester's local John build rather than in the hash or the target. A
purpose-built PDF cracker was therefore substituted.

> **Note for the reader:** this section is retained deliberately. Diagnosing and
> documenting a tooling failure — rather than silently working around it — is a core
> professional competency in penetration testing, and it demonstrates that the finding is
> reproducible rather than tool-dependent.

#### 3.2.4 Dictionary Attack

`pdfcrack` was run against each file using the `rockyou.txt` wordlist (~14.3 million
real-world leaked passwords):

```bash
pdfcrack -f report1.pdf -w /usr/share/wordlists/rockyou.txt
```

![pdfcrack results — reports 1 and 2](screenshots/10-pdfcrack-report1-2.png)

![pdfcrack results — report 3](screenshots/10b-pdfcrack-report3.png)

All three passwords were recovered almost immediately:

| File | Patient | Encryption | Recovered Password |
|---|---|---|---|
| `report1.pdf` | S. Dlamini | RC4-128 (R3) | `123456` |
| `report2.pdf` | P. Reddy | RC4-128 (R3) | `password` |
| `report3.pdf` | E. Thompson | RC4-128 (R3) | `!@#$%^&` |

Every recovered password is either a top-tier entry in every password-cracking wordlist or
a simple keyboard pattern. None offered meaningful resistance.

#### 3.2.5 Decryption

The recovered passwords were used to strip the encryption:

```bash
qpdf --password=123456  --decrypt report1.pdf report1_decrypted.pdf
qpdf --password=password --decrypt report2.pdf report2_decrypted.pdf
qpdf --password='!@#$%^&' --decrypt report3.pdf report3_decrypted.pdf
```

![Decryption](screenshots/11-qpdf-decrypt.png)

The contents were verified as genuine, readable patient medical records:

![Decrypted patient data](screenshots/11b-decrypted-patient-data.png)

> **MILESTONE 2 DELIVERABLE ACHIEVED** — the encryption on all three files was defeated and
> their full contents recovered, with proof of successful access.

### 3.3 Milestone 3 — Attack (Cracking): Critical Data Exposure

The assignment directed the tester to *"look beyond the obvious content — examine all file
properties carefully"*. Accordingly, the hidden metadata of each decrypted report was
examined with `exiftool`.

#### 3.3.1 Metadata Analysis

```bash
exiftool report3_decrypted.pdf
```

![Metadata — report 1](screenshots/12-pdf-metadata-report1.png)

![Metadata — report 2](screenshots/12b-pdf-metadata-report2.png)

![Metadata — report 3 (key finding)](screenshots/12c-pdf-metadata-report3-KEY.png)

The third report's metadata contained two disclosures of significant value:

```
Author   : j.malik
Comments : DB backup moved to /old before site migration, do not delete
```

1. **`j.malik`** — the internal username of the hospital's IT Systems Administrator,
   leaked directly into a document distributed to patients. This is a directly usable
   credential fragment for targeted attacks.
2. **The comment field** — an internal operational note that explicitly identifies the
   location of a database backup on the server: `/old/`.

#### 3.3.2 Tracing the Exposure

The metadata pointer corresponded exactly to the directory identified during
reconnaissance. The backup was retrieved without any authentication whatsoever:

```bash
curl -s https://medirozahospital.com/old/mediroza_db_backup_2019.sql \
     -o mediroza_db_backup_2019.sql
```

![SQL backup retrieved](screenshots/13-sql-backup-downloaded.png)

The file's own header confirms its sensitivity:

```
-- Mediroza General Hospital - internal database backup
-- Host: localhost    Database: mediroza_hr
-- WARNING: contains confidential staff and shareholder records
```

#### 3.3.3 Employee Salary Data Recovered

The backup contained a complete `staff` table of **30 employees**, including full name, job
title, department, email address, telephone number, **national identity number** and
**monthly salary**.

![Shareholder data](screenshots/14b-shareholders-data.png)

The complete register is reproduced in **Appendix A**. Summary figures:

| Metric | Value |
|---|---|
| Employees exposed | 30 |
| Total monthly payroll exposed | ZAR 2,027,000 |
| Total annual payroll exposed | ZAR 24,324,000 |
| National ID numbers exposed | 30 |
| Highest individual salary | Dr. Johan van der Merwe (Medical Director) — ZAR 160,000/month |

#### 3.3.4 Shareholder Data Recovered

The backup additionally contained a complete `shareholders` table of **10 entries**,
documenting the ownership structure of the hospital:

| Shareholder | Share % | Shares Held | Class |
|---|---|---|---|
| Dr. Rajesh Naidoo | 18.0 | 180,000 | Ordinary |
| Cedar Health Holdings (Pty) Ltd | 15.0 | 150,000 | Ordinary |
| Dr. Johan van der Merwe | 12.0 | 120,000 | Ordinary |
| Reddy Family Trust | 11.0 | 110,000 | Ordinary |
| Thabo Molefe | 10.0 | 100,000 | Ordinary |
| Sarah Botha | 9.0 | 90,000 | Ordinary |
| Dr. Ahmed Kara | 8.0 | 80,000 | Preferential |
| Naledi Zulu | 7.0 | 70,000 | Ordinary |
| Michael Roberts | 6.0 | 60,000 | Ordinary |
| Dr. Vikram Chetty | 4.0 | 40,000 | Preferential |

The complete register is reproduced in **Appendix B**.

> **MILESTONE 3 DELIVERABLE ACHIEVED** — full documented evidence of the exposure, together
> with a readable summary of the confidential data uncovered.

---

## 04 — Risk Rating

Each finding is rated using the standard Critical / High / Medium / Low scale, with
justification based on exploitability and business impact.

### V-01 — SQL Injection (Authentication Bypass)

| Attribute | Assessment |
|---|---|
| **Severity** | **CRITICAL** |
| **Location** | `https://medirozahospital.com/patient/login.php` |
| **Exploitability** | Trivial — a single unauthenticated HTTP request; no tooling required |
| **Impact** | Complete bypass of authentication; access to all patient records |
| **CVSS 3.1 (est.)** | 9.8 Critical |

**Justification.** The flaw requires no privileges, no user interaction and no specialist
knowledge. It grants access to the entire patient record set, which in a healthcare context
constitutes a direct breach of patient confidentiality and of medical data protection
obligations. The combination of trivial exploitability and maximum data sensitivity places
this finding at the top of the scale.

### V-02 — Exposed Database Backup

| Attribute | Assessment |
|---|---|
| **Severity** | **CRITICAL** |
| **Location** | `https://medirozahospital.com/old/mediroza_db_backup_2019.sql` |
| **Exploitability** | Trivial — a single unauthenticated HTTP GET |
| **Impact** | Disclosure of employee payroll, national ID numbers and corporate ownership |
| **CVSS 3.1 (est.)** | 9.1 Critical |

**Justification.** A complete database dump is downloadable by anyone. It exposes 30
national identity numbers, the full payroll structure and the hospital's shareholder
register. Beyond the confidentiality breach, national ID numbers enable identity fraud,
and payroll/shareholding data creates serious corporate and personal security risk. The
fact that the file was retained deliberately ("do not delete") indicates it is a known and
long-standing exposure rather than an accident.

### V-03 — Weak Passwords on Encrypted Patient Reports

| Attribute | Assessment |
|---|---|
| **Severity** | **HIGH** |
| **Location** | All three patient report PDFs served from `/patient/download.php` |
| **Exploitability** | Low — dictionary attack recovers passwords in seconds |
| **Impact** | Complete defeat of the only control protecting patient medical data |
| **CVSS 3.1 (est.)** | 7.5 High |

**Justification.** Although encryption was correctly applied, its protective value was
entirely negated by the use of `123456`, `password` and a keyboard pattern. The RC4-128
cipher itself is also deprecated and cryptographically weak. Encryption that can be removed
in seconds provides effectively no protection.

### V-04 — Directory Listing Enabled

| Attribute | Assessment |
|---|---|
| **Severity** | **HIGH** |
| **Location** | `/old/` and `/patient/` |
| **Exploitability** | Trivial — direct HTTP request |
| **Impact** | Full disclosure of server file structure and filenames |
| **CVSS 3.1 (est.)** | 7.5 High |

**Justification.** Directory listing converted what should have been an obscure filename into
a directly discoverable one. It was the enabling step that made the database backup trivial
to locate, and it exposed the internal structure of the patient portal to any visitor.

### V-05 — Information Disclosure via Document Metadata

| Attribute | Assessment |
|---|---|
| **Severity** | **MEDIUM** |
| **Location** | `/Author` and `/Comments` fields in report 3 |
| **Exploitability** | Low — requires only a metadata reader such as `exiftool` |
| **Impact** | Leakage of an internal admin username and an internal server path |
| **CVSS 3.1 (est.)** | 5.3 Medium |

**Justification.** Rated Medium in isolation because the metadata alone does not grant
access. However, in this engagement it was the **decisive pivot** — the comment field was
what connected the patient portal compromise to the far more severe database exposure. It
also leaks a valid username usable in credential-stuffing attacks. This finding should be
read in conjunction with V-02.

### Risk Summary

| ID | Finding | Severity |
|---|---|---|
| V-01 | SQL Injection — Authentication Bypass | **Critical** |
| V-02 | Exposed Database Backup (PII, payroll, shareholders) | **Critical** |
| V-03 | Weak Passwords on Encrypted Patient Reports | **High** |
| V-04 | Directory Listing Enabled | **High** |
| V-05 | Information Disclosure via PDF Metadata | **Medium** |

---

## 05 — Recommendations and Remediation

Recommendations are ordered by priority. Those marked **Immediate** should be actioned
before any other activity, as they address the two Critical findings.

### V-01 — SQL Injection  *(Immediate)*

1. **Use parameterised queries / prepared statements** for every database interaction.
   This is the definitive fix and eliminates the entire class of vulnerability. In PHP with
   MySQLi:

```php
// VULNERABLE
   $sql = "SELECT * FROM users WHERE username = '$username'";

// SECURE
$stmt = $conn->prepare("SELECT * FROM users WHERE username = ?");
$stmt->bind_param("s", $username);
$stmt->execute();
```

2. **Apply strict input validation** — reject any input containing quotes, semicolons or
   SQL comment sequences before it reaches the database layer.
3. **Suppress verbose database errors** in production. Errors must never be echoed to the
   user; log them server-side instead.
4. **Rotate all credentials** and review access logs for evidence of prior exploitation,
   since this flaw may have been abused before it was reported.

### V-02 — Exposed Database Backup  *(Immediate)*

1. **Remove the backup file from the web root immediately.** It must not be reachable over
   HTTP under any circumstances.
2. **Move all backups to non-web-accessible storage** — a separate host, object storage, or
   at minimum a directory outside the document root protected by server-level access rules.
3. **Encrypt backups at rest** using a strong, modern algorithm (AES-256) with keys held in
   a secrets manager, never on the same system as the backup.
4. **Treat the exposure as a data breach.** Because 30 national identity numbers were
   disclosed, the hospital should follow its statutory breach-notification obligations and
   consider notifying affected employees and the relevant regulator.
5. **Implement backup retention and disposal policies** so that superseded backups are
   destroyed on a defined schedule rather than retained indefinitely.

### V-03 — Weak Passwords on Encrypted Reports  *(High priority)*

1. **Eliminate password-protected PDFs as a delivery mechanism.** Passwords should not be
   transmitted to patients at all. Move to an authenticated portal where reports are
   delivered only after a successful login, protected by transport-layer security.
2. **If encryption must be retained**, generate high-entropy random passwords (minimum 16
   characters) rather than human-chosen ones, and deliver them through a separate channel.
3. **Upgrade the cipher** — RC4 is deprecated and must not be used. Adopt AES-256.
4. **Apply a password policy** for any staff-facing system, enforcing length and complexity
   and screening candidates against known-breached password lists.

### V-04 — Directory Listing  *(High priority)*

1. **Disable automatic directory indexing** on the web server. For LiteSpeed/Apache, ensure
   no `Indexes` option is present in the relevant `<Directory>` block, or place an
   `index.html` in each directory.
2. **Deny HTTP access to non-public directories** — specifically `/old/`, `/patient/reports/`
   and any backup, configuration or log paths.
3. **Audit the full web root** for further unintended exposure, as the same
   misconfiguration may affect paths not reached during this engagement.

### V-05 — Metadata Disclosure  *(Medium priority)*

1. **Strip document metadata before publication.** Remove `Author`, `Comments`, `Producer`
   and `Creator` fields from every PDF generated for external distribution.
2. **Automate sanitisation** in the report-generation pipeline so that metadata is cleaned
   by default rather than by manual intervention.
3. **Remove internal usernames and paths from all outbound documents**, and train staff
   that document properties are visible to recipients.
4. **Prefer generic service accounts** (for example `lab-reports@`) over individual staff
   usernames in the `Author` field of generated documents.

### Additional Strategic Recommendations

1. **Commission regular penetration testing** — this entire attack chain would have been
   identified by a routine assessment, and all findings are inexpensive to remediate.
2. **Introduce a web application firewall** as a compensating control while the code fixes
   are developed and deployed.
3. **Implement centralised logging and alerting** for authentication failures, unusual
   download volumes and access to sensitive paths.
4. **Adopt a secure development lifecycle** with code review and security testing embedded
   before release, particularly for any component handling patient data.

---

## Appendix A — Complete Staff Salary Register (30 Employees)

Recovered from `mediroza_db_backup_2019.sql`, table `staff`. Reproduced in full as
required by the Milestone 3 deliverable.

| # | Name | Job Title | Department | Monthly Salary (ZAR) | Annual (ZAR) |
|---|---|---|---|---|---|
| 3 | Dr. Johan van der Merwe | Medical Director | Management | 160,000 | 1,920,000 |
| 2 | Sarah Botha | Chief Financial Officer | Finance | 152,000 | 1,824,000 |
| 1 | Dr. Rajesh Naidoo | Chief Pathologist | Diagnostics Lab | 138,000 | 1,656,000 |
| 24 | Dr. Vikram Chetty | Anaesthetist | Theatre | 135,000 | 1,620,000 |
| 4 | Dr. Anita Naicker | Consultant Cardiologist | Cardiology | 132,000 | 1,584,000 |
| 21 | Dr. Suresh Moodley | Consultant Radiologist | Radiology | 130,000 | 1,560,000 |
| 5 | Dr. Ahmed Kara | Consultant Physician | Internal Medicine | 128,000 | 1,536,000 |
| 22 | Dr. Fatima Patel | Pediatrician | Pediatrics | 118,000 | 1,416,000 |
| 7 | Michael Roberts | HR Director | Human Resources | 96,000 | 1,152,000 |
| 6 | Dr. Yusuf Cassim | Senior Registrar | Emergency & Trauma | 74,000 | 888,000 |
| 15 | Kagiso Sithole | Pharmacist | Pharmacy | 61,000 | 732,000 |
| 9 | Jameel Malik | IT Systems Administrator | IT | 58,000 | 696,000 |
| 25 | David Smith | Facilities Manager | Operations | 52,000 | 624,000 |
| 23 | Nisha Singh | Physiotherapist | Rehabilitation | 48,000 | 576,000 |
| 10 | Thabo Molefe | Network Engineer | IT | 46,000 | 552,000 |
| 17 | Themba Nkosi | Radiographer | Radiology | 44,000 | 528,000 |
| 18 | Palesa Radebe | Radiographer | Radiology | 43,000 | 516,000 |
| 14 | Zanele Mahlangu | Nursing Sister | Theatre | 42,000 | 504,000 |
| 19 | Deepak Pillay | Lab Technologist | Diagnostics Lab | 41,000 | 492,000 |
| 29 | Peter van Wyk | Procurement Officer | Supply Chain | 38,000 | 456,000 |
| 13 | Bongani Ndlovu | Registered Nurse | Cardiology | 35,000 | 420,000 |
| 20 | Kavitha Govender | Lab Technician | Diagnostics Lab | 35,000 | 420,000 |
| 11 | Nomvula Khumalo | Registered Nurse | Emergency & Trauma | 34,000 | 408,000 |
| 12 | Lerato Mokoena | Registered Nurse | Pediatrics | 33,000 | 396,000 |
| 8 | Susan Pretorius | HR Officer | Human Resources | 32,000 | 384,000 |
| 26 | Karen O'Connor | Billing Administrator | Finance | 29,000 | 348,000 |
| 27 | James Wilson | Security Supervisor | Operations | 27,000 | 324,000 |
| 16 | Naledi Zulu | Pharmacy Assistant | Pharmacy | 26,000 | 312,000 |
| 30 | Andile Mbeki | Ward Clerk | Administration | 21,000 | 252,000 |
| 28 | Linda Fourie | Receptionist | Front Office | 19,000 | 228,000 |

**Totals:** monthly payroll ZAR 2,027,000 · annual payroll ZAR 24,324,000

### Additional Personal Data Exposed

Beyond salary, each record also contained the following, all of which are sensitive:

| Data Element | Sensitivity |
|---|---|
| National identity number | Enables identity fraud; regulated personal data |
| Personal email address | Target for phishing and credential stuffing |
| Personal telephone number | Target for social engineering and SIM-swap attacks |

A representative sample (deliberately truncated) illustrates the exposure:

| Name | Email | Phone | National ID |
|---|---|---|---|
| Dr. Johan van der Merwe | j.merwe@medirozahospital.com | +27 82 103 2021 | 85041033033083 |
| Sarah Botha | s.botha@medirozahospital.com | +27 82 102 2014 | 85031023022082 |
| Dr. Rajesh Naidoo | r.naidoo@medirozahospital.com | +27 82 101 2007 | 85021013011081 |
| Dr. Vikram Chetty | v.chetty@medirozahospital.com | +27 82 124 2168 | 85011243264086 |
| Dr. Anita Naicker | a.naicker@medirozahospital.com | +27 82 104 2028 | 85051043044084 |

The remaining 25 records follow the identical structure and are held in the accompanying
evidence file `evidence/staff_salaries.csv`.

---

## Appendix B — Complete Shareholder Register (10 Entries)

Recovered from `mediroza_db_backup_2019.sql`, table `shareholders`.

| # | Shareholder | Share % | Shares Held | Share Class |
|---|---|---|---|---|
| 1 | Dr. Rajesh Naidoo | 18.0 | 180,000 | Ordinary |
| 2 | Cedar Health Holdings (Pty) Ltd | 15.0 | 150,000 | Ordinary |
| 3 | Dr. Johan van der Merwe | 12.0 | 120,000 | Ordinary |
| 4 | Reddy Family Trust | 11.0 | 110,000 | Ordinary |
| 5 | Thabo Molefe | 10.0 | 100,000 | Ordinary |
| 6 | Sarah Botha | 9.0 | 90,000 | Ordinary |
| 7 | Dr. Ahmed Kara | 8.0 | 80,000 | Preferential |
| 8 | Naledi Zulu | 7.0 | 70,000 | Ordinary |
| 9 | Michael Roberts | 6.0 | 60,000 | Ordinary |
| 10 | Dr. Vikram Chetty | 4.0 | 40,000 | Preferential |

**Total shares accounted for:** 1,000,000 (100% of issued capital).

---

## Appendix C — Patient Data Recovered

Three confidential pathology reports were retrieved and decrypted in full. The following
summarises the personal and medical data exposed in each.

### Report 1 — Sipho Dlamini

| Field | Value |
|---|---|
| Patient ID | MG-P-10231 |
| Date of Birth | 1984-06-12 |
| Gender | Male |
| Report Date | 2024-11-04 |
| Referring Doctor | Dr. Anita Naicker |
| Lab Reference | LR-2024-1187 |
| Specimen | Whole blood (EDTA) |
| Test Panel | Full Blood Count |
| Abnormal Results | White Cell Count 11.8 (ref 4.0–11.0) — HIGH |

### Report 2 — Priya Reddy

| Field | Value |
|---|---|
| Patient ID | MG-P-10244 |
| Date of Birth | 1991-02-28 |
| Gender | Female |
| Report Date | 2024-11-05 |
| Referring Doctor | Dr. Johan van der Merwe |
| Lab Reference | LR-2024-1192 |
| Specimen | Serum |
| Test Panel | Lipid Profile |
| Abnormal Results | Total Cholesterol 6.0 (ref <5.0) HIGH; LDL 4.1 (ref <3.0) HIGH; Triglycerides 1.8 (ref <1.7) HIGH |

### Report 3 — Emily Thompson

| Field | Value |
|---|---|
| Patient ID | MG-P-10258 |
| Date of Birth | 1978-09-03 |
| Gender | Female |
| Report Date | 2024-11-06 |
| Referring Doctor | Dr. Ahmed Kara |
| Lab Reference | LR-2024-1205 |
| Specimen | Serum |
| Test Panel | Full Blood Count |
| Abnormal Results | Haemoglobin 11.4 (ref 12.0–15.5) LOW; Ferritin 9 (ref 15–150) LOW; Vitamin D 42 (ref 50–125) LOW |

Each report also disclosed the **reporting pathologist**, the laboratory's SANAS
accreditation reference, and the hospital's internal CMS version (`Mediroza CMS 1.4.2`) —
the latter being directly useful to an attacker selecting version-specific exploits.

---

## Appendix D — Evidence Index

| # | Screenshot | Milestone | Description |
|---|---|---|---|
| 01 | `recon-robots.png` | M1 | robots.txt disclosing /patient/, /staff/, /old/ |
| 02 | `directory-listing.png` | M1 | Directory listing on /old/ and /patient/ |
| 03 | `login-page.png` | M1 | Patient portal login form and input fields |
| 04 | `sqli-auth-bypass.png` | M1 | SQL injection — HTTP 302 to portal.php |
| 05 | `portal-reports.png` | M1 | Three confidential reports enumerated in portal |
| 06 | `downloaded-pdfs.png` | M1 | All three encrypted PDFs retrieved |
| 07 | `pdf-encryption-info.png` | M2 | qpdf --show-encryption — reports 1 and 2 |
| 07b | `pdf-encryption-report3.png` | M2 | qpdf --show-encryption — report 3 |
| 08 | `pdf2john-hashes.png` | M2 | Extracted $pdf$ hash |
| 09 | `john-error-load.png` | M2 | John load failure — initial diagnosis |
| 09b | `john-still-failing.png` | M2 | Corrected hash still failing — build limitation |
| 10 | `pdfcrack-report1-2.png` | M2 | Passwords recovered for reports 1 and 2 |
| 10b | `pdfcrack-report3.png` | M2 | Password recovered for report 3 |
| 11 | `qpdf-decrypt.png` | M2 | Decryption of all three reports |
| 11b | `decrypted-patient-data.png` | M2 | Readable patient data confirmed |
| 12 | `pdf-metadata-report1.png` | M3 | exiftool metadata — report 1 |
| 12b | `pdf-metadata-report2.png` | M3 | exiftool metadata — report 2 |
| 12c | `pdf-metadata-report3-KEY.png` | M3 | exiftool metadata — report 3 (key finding) |
| 13 | `sql-backup-downloaded.png` | M3 | Database backup retrieved |
| 14b | `shareholders-data.png` | M3 | Shareholder register extracted |

---

## Appendix E — Milestone Completion Summary

| Milestone | Objective | Status |
|---|---|---|
| **M1** | Attack the website and retrieve 3 confidential patient PDF lab reports | ✅ Complete |
| **M2** | Crack the encryption on all 3 retrieved files | ✅ Complete |
| **M3** | Find staff salaries and shareholder details of the hospital | ✅ Complete |
| **M4** | Write a professional penetration testing report | ✅ Complete |

---

## Statement of Authorisation

This penetration test was performed against `https://medirozahospital.com` under written
authorisation provided by NetworkWalks for the purposes of the Batch B083 Week 4 security
training programme. The target was explicitly designated as authorised for security
testing.

All testing was conducted within the agreed scope and rules of engagement. No social
engineering was performed, no denial-of-service testing was conducted, and no system
outside the authorised target was touched. The techniques documented in this report are
presented for defensive and educational purposes and must never be applied to any system
without explicit written permission from its owner.

---

**Report prepared by:** Shiv Kumar Das  
**Programme:** NetworkWalks — Batch B083, Week 4  
**Date:** 3 October 2026  
**Classification:** CONFIDENTIAL — Authorised Personnel Only
