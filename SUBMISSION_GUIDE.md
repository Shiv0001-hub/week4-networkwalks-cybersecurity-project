# Submission Guide — GitHub + LinkedIn

**Project:** Mediroza General Hospital Penetration Test · NetworkWalks Batch B083 Week 4
**Author:** Shiv Kumar Das

---

## STEP A — Move the files to your Kali machine

The ZIP file (`mediroza-pentest-submission.zip`) has been provided. Download it, then:

```bash
cd ~/mediroza
unzip ~/Downloads/mediroza-pentest-submission.zip
```

You now have a folder `mediroza-submission/` containing:

```
mediroza-submission/
├── README.md
├── Mediroza_Penetration_Test_Report.docx     <- submit this to your instructor
├── Mediroza_Penetration_Test_Report.md
├── LINKEDIN_POST.md
├── screenshots/                              <- 22 evidence screenshots
└── evidence/
    ├── mediroza_db_backup_2019.sql
    ├── staff_salaries.csv
    ├── shareholders.csv
    └── patient_reports/ (6 PDFs)
```

---

## STEP B — Create the GitHub repository

1. Go to https://github.com and log in
2. Click **+** (top right) -> **New repository**
3. Settings:
   - Repository name: `mediroza-pentest-report`
   - Description: `Black-box Penetration Testing Report - Mediroza General Hospital | NetworkWalks Batch B083 Week 4`
   - Visibility: **Public**
   - Do NOT tick "Add a README file"
4. Click **Create repository**
5. Keep the page open - you need the URL

---

## STEP C — Push from Kali

Replace `YOUR-USERNAME` with your real GitHub username.

```bash
cd ~/mediroza/mediroza-submission
```

```bash
git init
```

```bash
git add .
```

```bash
git commit -m "Mediroza General Hospital - Penetration Testing Report (NetworkWalks B083 WK4)"
```

```bash
git branch -M main
```

```bash
git remote add origin https://github.com/YOUR-USERNAME/mediroza-pentest-report.git
```

```bash
git push -u origin main
```

### If it asks for a password

GitHub no longer accepts your account password. Create a **Personal Access Token**:

1. GitHub -> **Settings** -> **Developer settings** -> **Personal access tokens** -> **Tokens (classic)**
2. **Generate new token (classic)**
3. Tick the **`repo`** scope
4. Expiry: 90 days
5. Copy the token (it starts with `ghp_`)
6. Paste it when `git push` asks for a password

Username = your GitHub username. Password = the token.

---

## STEP D — Verify

Refresh the repository page. You should see:
- The README rendered with the findings table
- `screenshots/` folder with 22 PNGs
- `evidence/` folder with the SQL backup, CSVs and PDFs

**Your submission link:**
```
https://github.com/YOUR-USERNAME/mediroza-pentest-report
```

---

## STEP E — LinkedIn post

Open `LINKEDIN_POST.md` and choose one of the three versions:

| Option | Best for |
|---|---|
| Option 1 - The Story | Best reach, walks through all 7 steps |
| Option 2 - Short & punchy | Quick scroll-stopper |
| Option 3 - Technical | Security-focused audience |

Posting checklist:

1. Copy the text of your chosen option
2. LinkedIn -> **Start a post** -> paste
3. Attach 3 screenshots (recommended):
   - `screenshots/04-sqli-auth-bypass.png`
   - `screenshots/12c-pdf-metadata-report3-KEY.png`
   - `screenshots/13-sql-backup-downloaded.png`
4. Publish the post
5. Add your GitHub link as the **first comment** (links in the post body reduce reach)
6. Tag your instructor and NetworkWalks

---

## Suggested message to your instructor

> Dear Sir,
>
> Please find my Week 4 penetration testing project for Mediroza General Hospital.
>
> GitHub repository (full report, evidence and screenshots):
> https://github.com/YOUR-USERNAME/mediroza-pentest-report
>
> The Word report is also attached for your review.
>
> All four milestones were completed: initial access via SQL injection, cracking the
> encryption on all three patient reports, and identifying the exposed database backup
> containing staff salaries and shareholder details.
>
> Regards,
> Shiv Kumar Das

---

## Quick reference — what was achieved

| Milestone | Objective | Status |
|---|---|---|
| M1 | Retrieve 3 confidential patient PDF lab reports | Complete |
| M2 | Crack the encryption on all 3 files | Complete |
| M3 | Find staff salaries and shareholder details | Complete |
| M4 | Write a professional penetration testing report | Complete |

**Findings:** 2 Critical, 2 High, 1 Medium.
