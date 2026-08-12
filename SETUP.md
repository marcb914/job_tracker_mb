# Job Alert Setup Guide

Three files go into your existing `job-tracker` GitHub repo. Once added, you'll get a daily email every weekday morning listing any new role openings detected.

---

## Files to add

```
job-tracker/
├── index.html              ← already there (your dashboard)
├── check_jobs.py           ← NEW: the checker script
├── job_state.json          ← NEW: auto-created on first run (leave empty for now)
└── .github/
    └── workflows/
        └── daily_job_check.yml   ← NEW: the automation schedule
```

---

## Step 1 — Generate a Gmail App Password

Google won't let scripts use your real password. You need a one-time App Password instead.

1. Go to **myaccount.google.com/security**
2. Under "How you sign in to Google," click **2-Step Verification** and make sure it's ON (required)
3. Go back to Security and search for **"App passwords"** (or go directly to myaccount.google.com/apppasswords)
4. Click **Create**, give it a name like `job-alert`, click **Create**
5. Google shows you a **16-character password** — copy it now, it won't be shown again

---

## Step 2 — Add secrets to your GitHub repo

1. Go to your `job-tracker` repo on GitHub
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** and add these two:

| Secret name | Value |
|---|---|
| `GMAIL_USER` | your full Gmail address (e.g. `marcb914@gmail.com`) |
| `GMAIL_APP_PASS` | the 16-character App Password from Step 1 |

---

## Step 3 — Upload the new files

In your repo on GitHub:

1. Upload `check_jobs.py` to the root (same folder as `index.html`)
2. Create a file called `job_state.json` containing just `{}` (two curly braces — this initializes the state)
3. Create `.github/workflows/daily_job_check.yml` — paste in the workflow file contents

To create nested folders on GitHub: when uploading a file, type `.github/workflows/daily_job_check.yml` as the filename — GitHub will create the folders automatically.

---

## Step 4 — Test it manually

1. Go to your repo → **Actions** tab
2. Click **Daily Job Check** in the left sidebar
3. Click **Run workflow** → **Run workflow**
4. Watch it run (takes ~60 seconds)
5. Check your Gmail — you should get a first-run email showing everything detected

After the first run, `job_state.json` will be committed back to your repo automatically. From then on, you only get emails when something *changes*.

---

## Schedule

Runs automatically at **8am Central Time, Monday–Friday**.

To change the time, edit the `cron` line in `daily_job_check.yml`:
- `"0 13 * * 1-5"` = 8am CT weekdays
- `"0 14 * * *"` = 9am CT every day including weekends
- `"0 13,19 * * 1-5"` = 8am and 2pm CT weekdays (twice daily)

---

## How it works

The script fetches each career page URL and looks for keywords that indicate the role is live (e.g. "associate product manager" on Databricks' careers page). It compares today's results to yesterday's saved state and emails you only when something changes — new opening detected, or a previously-detected role disappears.

**Important caveat:** Some career pages (Google, Meta, Salesforce) load jobs via JavaScript, which means the script sees minimal HTML. These may produce false negatives — the role is live but the script doesn't detect it. For these, APMList and Simplify are still your best real-time sources. The script works most reliably on Greenhouse-hosted pages (ClearView, Roivant, Veeva) and simpler career pages.

---

## To add or remove companies

Edit `check_jobs.py` and find the `TARGETS` list. Each entry is:

```python
("Label shown in email",
 "https://careers.company.com/page",
 ["keyword1", "keyword2"]),
```

Add a new tuple, save, re-upload the file to GitHub.

