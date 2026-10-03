# LinkedIn Post — Ready to Copy & Paste

> **How to use:** Post option 1 (the story) — it performs best. Attach 2–4 screenshots from
> `screenshots/` (best picks: `04-sqli-auth-bypass.png`, `12c-pdf-metadata-report3-KEY.png`,
> `13-sql-backup-downloaded.png`). Add the repo link in the comments or first comment.
> Suggested hashtags are included at the end of each option.

---

## Option 1 — The Story (recommended)

I was given written permission to attack a hospital's website.

So I did. And I got in.

Here's the full chain, step by step 👇

**Step 1 — Recon**
The site's `robots.txt` told search engines to ignore three folders: `/patient/`, `/staff/`
and `/old/`. Ironically, that's a map of exactly where the sensitive stuff lives.

**Step 2 — Directory listing**
Two of those folders had directory listing switched on. I could see every filename — including
`mediroza_db_backup_2019.sql`.

**Step 3 — Authentication bypass**
The patient portal login form took a username and password. I sent this:

    username = admin'--      password = anything

One quote closes the SQL string. Two dashes comment out the password check.
The server replied `HTTP 302 → portal.php` and handed me a valid session.

I never had a password. I never needed one.

**Step 4 — Three confidential patient reports**
Behind the login sat three encrypted pathology lab reports. I downloaded all of them.

**Step 5 — Cracking the encryption**
All three used RC4-128 encryption. The encryption method was identical — the only difference
was password strength. I ran them against a wordlist of 14 million leaked passwords:

    report1 → 123456
    report2 → password
    report3 → !@#$%^&

Seconds. All three. Confidential patient medical records, fully readable.

**Step 6 — The pivot**
The assignment said to look at the file properties. So I ran `exiftool` on the third report
and found this in the metadata:

    Author   : j.malik
    Comments : DB backup moved to /old before site migration, do not delete

A note left behind by the hospital's own IT admin. It pointed straight at that database
backup I'd spotted in Step 2.

**Step 7 — What was actually exposed**
I downloaded it. No authentication required. It contained:

→ The salaries of all 30 hospital employees
→ 30 national identity numbers
→ Every employee's email, phone number and job title
→ The complete shareholder register — who owns the hospital

Total payroll exposed: ZAR 24.3 million per year.

---

**What I took away from this:**

The SQL injection wasn't the most interesting part. The most interesting part was that the
encryption *worked* — the PDFs were genuinely encrypted with RC4-128. It just didn't matter,
because the password was `123456`.

Security controls that exist on paper but not in practice are worse than no controls at all,
because they create the *impression* of safety.

And the metadata finding is the one I'll remember: the breach wasn't completed by a clever
exploit. It was completed by a comment field someone forgot to clean up.

Full report with all evidence is on GitHub (link in comments).

---

🔒 This was an authorised, educational penetration test conducted under the NetworkWalks
training programme. Everything here was done with written permission, against a target
explicitly designated for testing.

`#CyberSecurity` `#PenetrationTesting` `#EthicalHacking` `#InfoSec` `#ApplicationSecurity`
`#KaliLinux` `#LearningInPublic`

---

## Option 2 — Short & punchy

I was given permission to attack a hospital website. Here's what happened:

🔍 robots.txt leaked 3 hidden folders
📂 Directory listing exposed a database backup
💉 SQL injection (`admin'--`) bypassed the login completely
📄 Downloaded 3 encrypted patient lab reports
🔓 Cracked all 3 passwords in seconds — `123456`, `password`, `!@#$%^&`
🕵️ Found an IT admin's note in the PDF metadata pointing to the backup
💰 Downloaded it: 30 employee salaries + the full shareholder register

Total payroll exposed: ZAR 24.3M/year.

The lesson? The encryption worked fine. The passwords were `123456`.

A control that only exists on paper is worse than no control at all.

Full report + evidence on GitHub 👇 (link in comments)

`#CyberSecurity` `#PenetrationTesting` `#EthicalHacking` `#InfoSec`

---

## Option 3 — Technical / for a security audience

Completed a full black-box pentest of a hospital web app. The attack chain:

1. `robots.txt` → `/patient/`, `/staff/`, `/old/`
2. Directory listing enabled on both `/old/` and `/patient/`
3. SQLi auth bypass on `/patient/login.php` — payload `admin'--`, any password →
   `302 → portal.php` + valid `PHPSESSID`
4. 3 patient PDFs retrieved via `download.php?id=N`
5. All 3 = Standard security handler, V2/R3, RC4-128 → `pdf2john` → `pdfcrack` + rockyou
6. Passwords: `123456`, `password`, `!@#$%^&`
7. `exiftool` on report 3 → `Author: j.malik`, `Comments: "DB backup moved to /old"`
8. `/old/mediroza_db_backup_2019.sql` → 30 staff records (salary, national ID, phone, email)
   + 10 shareholder records

Two Critical findings: SQLi auth bypass (CVSS ~9.8) and an exposed DB backup with PII and
payroll data (CVSS ~9.1).

Worth noting: `john --format=PDF` refused to load the extracted hashes — a known pdf2john
defect (openwall/john#4739) where the `/P` permissions field is serialised as an unsigned
value. Verified the hash against both current and legacy `pdf_valid()` parsers, confirmed it
sound, and switched to `pdfcrack`. Documented in the report rather than silently worked
around.

Full report with all 22 evidence screenshots on GitHub.

`#CyberSecurity` `#PenetrationTesting` `#InfoSec` `#AppSec` `#CVSS`

---

## 📸 Screenshot pairing suggestions

| Screenshot | Why it works |
|---|---|
| `04-sqli-auth-bypass.png` | The `302 → portal.php` moment — the "I'm in" shot |
| `12c-pdf-metadata-report3-KEY.png` | The metadata comment that cracked the case open |
| `13-sql-backup-downloaded.png` | "WARNING: contains confidential staff and shareholder records" |
| `10-pdfcrack-report1-2.png` | Passwords recovered: `123456`, `password` |
| `05-portal-reports.png` | Three patient names visible in the portal |

**Tip:** Don't post more than 4 images — LinkedIn reach drops. Pick the 3 that tell the story.
