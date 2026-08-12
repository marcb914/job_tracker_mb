"""
New Grad Job Alert — Marc Betancourt 2027
Runs daily via GitHub Actions. Checks career pages for new postings,
compares against last run, emails a digest if anything changed.
"""

import os, json, smtplib, datetime, urllib.request, urllib.error
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

# ── CONFIG (set via GitHub Secrets) ──────────────────────────────────────────
GMAIL_USER   = os.environ["GMAIL_USER"]       # your Gmail address
GMAIL_PASS   = os.environ["GMAIL_APP_PASS"]   # Gmail App Password (not your login password)
ALERT_TO     = os.environ.get("ALERT_TO", GMAIL_USER)  # defaults to same address
STATE_FILE   = "job_state.json"               # persisted in your repo between runs

# ── TARGETS ──────────────────────────────────────────────────────────────────
# Each entry: (label, url, keywords_that_indicate_role_is_live)
# We fetch the page and look for the keywords. If they appear and weren't
# there yesterday, we flag it as newly opened.
TARGETS = [
    # --- Tech APM ---
    ("Databricks APM",
     "https://www.databricks.com/company/careers/university-recruiting",
     ["product manager", "associate product manager", "apm"]),

    ("Google APM",
     "https://www.google.com/about/careers/applications/jobs/results?category=PRODUCT_MANAGEMENT&employment_type=FULL_TIME&degree=BACHELORS",
     ["associate product manager", "apm"]),

    ("Meta RPM",
     "https://www.metacareers.com/jobs?teams%5B0%5D=Internship%20-%20Engineering%2C%20Tech%20%26%20Design&teams%5B1%5D=Internship%20-%20Business&q=rotational+product+manager",
     ["rotational product manager", "rpm"]),

    ("Salesforce APM",
     "https://salesforce.wd12.myworkdayjobs.com/en-US/External_Career_Site/jobs?q=associate+product+manager&locations=All+Locations",
     ["associate product manager"]),

    ("Intuit RPM",
     "https://careers.intuit.com/job-search?keyword=rotational+product+manager&category=Product+Management",
     ["rotational product manager", "rpm"]),

    ("Atlassian APM",
     "https://www.atlassian.com/company/careers/all-jobs?team=Product+Management&search=associate",
     ["associate product manager"]),

    ("Microsoft PM New Grad",
     "https://jobs.careers.microsoft.com/global/en/search?q=product+manager&lc=United+States&exp=Students+and+recent+graduates",
     ["product manager", "new grad", "university"]),

    ("LinkedIn APB",
     "https://careers.linkedin.com/jobs?keywords=associate+product&location=United+States",
     ["associate product builder", "apb", "associate product manager"]),

    ("Capital One APM",
     "https://www.capitalonecareers.com/search-jobs/associate%20product%20manager/4973/1",
     ["associate product manager"]),

    # --- Life Sciences Consulting ---
    ("ZS Associates DAA",
     "https://www.zs.com/careers/find-a-job?area=Business+Consulting+%26+Technology&type=Full-Time&experience=Entry+Level",
     ["decision analytics associate", "business consulting associate", "daa", "bca"]),

    ("ClearView Healthcare",
     "https://boards.greenhouse.io/clearviewhealthcarepartners",
     ["analyst", "life sciences", "strategy"]),

    ("Trinity Life Sciences",
     "https://www.trinitylifesciences.com/careers/",
     ["analyst", "associate"]),

    ("Analysis Group",
     "https://www.analysisgroup.com/careers/experienced-professionals/open-positions/?practice=health-care-life-sciences",
     ["analyst", "health", "life sciences"]),

    # --- Biotech / Pharma ---
    ("Genentech CRDP",
     "https://www.gene.com/careers/detail?requisitionId=202212-126004",
     ["commercial rotation", "crdp", "rotation development"]),

    ("Roivant Rotational Analyst",
     "https://boards.greenhouse.io/roivantsciences",
     ["rotational analyst", "analyst program", "rotation"]),

    ("AstraZeneca CLDP",
     "https://careers.astrazeneca.com/job-search?keyword=leadership+development&country=United+States&category=Commercial",
     ["commercial leadership", "cldp", "leadership development"]),

    ("J&J FLDP",
     "https://jobs.jnj.com/jobs?keywords=finance+leadership+development&location=United+States",
     ["finance leadership development", "fldp"]),

    # --- Analytics & Strategy ---
    ("Amazon BA / BIE New Grad",
     "https://www.amazon.jobs/en/search?base_query=business+analyst&category%5B%5D=business-and-merchant-development&job_type%5B%5D=Full-Time&experience_ids%5B%5D=entry-level",
     ["business analyst", "business intelligence engineer", "new grad", "entry level"]),

    ("Visa NCG Analyst",
     "https://jobs.smartrecruiters.com/Visa/743999941123456-associate-business-analyst",
     ["associate business analyst", "new college grad", "ncg"]),

    ("DaVita Redwoods",
     "https://jobs.davita.com/search-jobs/redwoods",
     ["redwoods", "analyst", "leadership"]),

    # --- Veeva ---
    ("Veeva Generation Veeva BCDP",
     "https://careers.veeva.com/job-search/?keyword=generation+veeva",
     ["generation veeva", "associate business consultant", "bcdp", "analytics development"]),
]

# ── HELPERS ───────────────────────────────────────────────────────────────────

def fetch_page(url: str) -> str:
    """
    Fetches a URL and returns lowercased text content.
    Returns empty string on any error so we skip gracefully.
    Many career pages are JS-rendered and will return minimal HTML —
    that's expected; we're looking for keyword presence in what IS returned.
    """
    req = urllib.request.Request(
        url,
        headers={"User-Agent": "Mozilla/5.0 (compatible; job-alert-bot/1.0)"}
    )
    try:
        with urllib.request.urlopen(req, timeout=15) as resp:
            return resp.read().decode("utf-8", errors="ignore").lower()
    except Exception as e:
        print(f"  Fetch error for {url}: {e}")
        return ""


def check_target(label: str, url: str, keywords: list[str]) -> bool:
    """
    Returns True if ANY keyword is found on the page.
    This is a positive-signal check — we assume the role is live
    if the keywords appear on the careers page.
    """
    content = fetch_page(url)
    if not content:
        return False
    return any(kw.lower() in content for kw in keywords)


def load_state() -> dict:
    """Loads yesterday's results from the JSON state file."""
    if os.path.exists(STATE_FILE):
        try:
            with open(STATE_FILE) as f:
                return json.load(f)
        except Exception:
            pass
    return {}


def save_state(state: dict):
    """Saves today's results to JSON so tomorrow can diff against it."""
    with open(STATE_FILE, "w") as f:
        json.dump(state, f, indent=2)


def build_email(new_open: list, still_open: list, newly_closed: list) -> tuple[str, str]:
    """
    Builds subject line and HTML body for the alert email.
    Only called when there is something worth reporting.
    """
    today = datetime.date.today().strftime("%B %d, %Y")
    subject = f"🚨 Job Alert — {len(new_open)} new opening(s) detected — {today}"

    rows_new = "".join(
        f"<tr style='background:#f0faf4'>"
        f"<td style='padding:10px 14px;font-weight:600;color:#057a55'>{label}</td>"
        f"<td style='padding:10px 14px'><a href='{url}' style='color:#1a56db'>{url}</a></td>"
        f"<td style='padding:10px 14px;color:#057a55;font-weight:600'>🟢 NEW</td>"
        f"</tr>"
        for label, url in new_open
    )
    rows_still = "".join(
        f"<tr>"
        f"<td style='padding:8px 14px;color:#374151'>{label}</td>"
        f"<td style='padding:8px 14px'><a href='{url}' style='color:#1a56db'>{url}</a></td>"
        f"<td style='padding:8px 14px;color:#b45309'>Still open</td>"
        f"</tr>"
        for label, url in still_open
    )
    rows_closed = "".join(
        f"<tr style='opacity:.6'>"
        f"<td style='padding:8px 14px;color:#6b7280'>{label}</td>"
        f"<td style='padding:8px 14px'><a href='{url}' style='color:#9ca3af'>{url}</a></td>"
        f"<td style='padding:8px 14px;color:#6b7280'>No longer detected</td>"
        f"</tr>"
        for label, url in newly_closed
    )

    html = f"""
    <html><body style="font-family:Inter,system-ui,sans-serif;color:#111;max-width:760px;margin:0 auto;padding:24px">
    <h1 style="font-size:20px;font-weight:600;margin-bottom:4px">New Grad Job Alert</h1>
    <p style="color:#6b7280;font-size:13px;margin-bottom:24px">{today} · automated daily check</p>

    {"<h2 style='font-size:15px;color:#057a55;margin-bottom:8px'>🟢 Newly detected ({len(new_open)})</h2><table width='100%' cellpadding='0' cellspacing='0' style='border:1px solid #e5e7eb;border-radius:8px;border-collapse:collapse;margin-bottom:24px;overflow:hidden'>" + rows_new + "</table>" if new_open else ""}
    {"<h2 style='font-size:15px;color:#b45309;margin-bottom:8px'>🟡 Still open from before</h2><table width='100%' cellpadding='0' cellspacing='0' style='border:1px solid #e5e7eb;border-radius:collapse;margin-bottom:24px'>" + rows_still + "</table>" if still_open else ""}
    {"<h2 style='font-size:15px;color:#6b7280;margin-bottom:8px'>⚫ No longer detected</h2><table width='100%' cellpadding='0' cellspacing='0' style='border:1px solid #e5e7eb;border-radius:8px;border-collapse:collapse;margin-bottom:24px'>" + rows_closed + "</table>" if newly_closed else ""}

    <p style="font-size:12px;color:#9ca3af;border-top:1px solid #f3f4f6;padding-top:16px;margin-top:24px">
    This is an automated check. Some career pages are JS-rendered and may show false negatives —
    always verify directly on the company site. Apply same day for short-window roles (Salesforce, Atlassian, Figma).
    </p>
    </body></html>
    """
    return subject, html


def send_email(subject: str, html: str):
    """Sends the alert email via Gmail SMTP with TLS."""
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"]    = GMAIL_USER
    msg["To"]      = ALERT_TO
    msg.attach(MIMEText(html, "html"))

    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
        server.login(GMAIL_USER, GMAIL_PASS)
        server.sendmail(GMAIL_USER, ALERT_TO, msg.as_string())
    print(f"Email sent to {ALERT_TO}")


# ── MAIN ──────────────────────────────────────────────────────────────────────

def main():
    print(f"Running job check — {datetime.datetime.now().isoformat()}")
    prev_state = load_state()
    curr_state = {}

    # Check every target
    for label, url, keywords in TARGETS:
        print(f"  Checking: {label}...")
        found = check_target(label, url, keywords)
        curr_state[label] = {"found": found, "url": url}
        print(f"    → {'FOUND' if found else 'not found'}")

    # Diff against yesterday
    new_open     = []   # was not found yesterday, found today
    still_open   = []   # was found yesterday, still found today
    newly_closed = []   # was found yesterday, not found today

    for label, info in curr_state.items():
        url = info["url"]
        found_today = info["found"]
        found_yesterday = prev_state.get(label, {}).get("found", False)

        if found_today and not found_yesterday:
            new_open.append((label, url))
        elif found_today and found_yesterday:
            still_open.append((label, url))
        elif not found_today and found_yesterday:
            newly_closed.append((label, url))

    # Save state for tomorrow's run
    save_state(curr_state)
    print(f"State saved. New: {len(new_open)}, Still open: {len(still_open)}, Closed: {len(newly_closed)}")

    # Only send email if something changed
    if new_open or newly_closed:
        print("Changes detected — sending alert email...")
        subject, html = build_email(new_open, still_open, newly_closed)
        send_email(subject, html)
    else:
        print("No changes detected. No email sent.")


if __name__ == "__main__":
    main()
