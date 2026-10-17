<div align="center">

<img src="expense-tracker/static/images/iskcon-shirpur-logo.jpeg" alt="ISKCON Shirpur" width="120"/>

# 🪷 ISKCON Shirpur Finance Management System

**Transparent, role-based temple finance — expenses, budgets, donations, alerts and audit trails in one place.**

![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-3.1-000000?logo=flask&logoColor=white)
![Postgres](https://img.shields.io/badge/Supabase-PostgreSQL-3ECF8E?logo=supabase&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-local_default-003B57?logo=sqlite&logoColor=white)
![Clerk](https://img.shields.io/badge/Auth-Clerk_(optional)-6C47FF?logo=clerk&logoColor=white)
![Twilio](https://img.shields.io/badge/WhatsApp-Twilio-F22F46?logo=twilio&logoColor=white)
![Tests](https://img.shields.io/badge/tests-pytest-0A9EDC?logo=pytest&logoColor=white)

*"Every rupee offered in service deserves to be accounted for with devotion."*

</div>

---

## 📖 Table of Contents

1. [About](#-about)
2. [Features](#-features)
3. [System Architecture](#-system-architecture)
4. [Project Flow](#-project-flow)
5. [Roles & Access Control](#-roles--access-control)
6. [Donation Pipeline](#-donation-pipeline)
7. [Background Jobs](#-background-jobs)
8. [Database Schema](#-database-schema)
9. [Project Structure](#-project-structure)
10. [Tech Stack](#-tech-stack)
11. [Getting Started](#-getting-started)
12. [Configuration](#-configuration)
13. [Testing](#-testing)
14. [Deployment](#-deployment)
15. [Security Notes](#-security-notes)
16. [Author](#-author)

---

## 🕉️ About

The ISKCON Shirpur Finance Management System replaces scattered registers and spreadsheets with a single web app for the temple's accounts team. It tracks **department-wise expenses with bill uploads**, enforces **monthly budgets**, records **donations** (and forwards them to an external system), lets the Temple Management Committee (TMC) **raise alerts and queries**, and keeps a full **activity log** of every change.

It runs out of the box on **SQLite** with zero setup, and switches to **Supabase/PostgreSQL**, **Clerk** sign-in, **Google Drive** backups, **Gmail** email and **WhatsApp** notifications purely by adding environment variables.

---

## ✨ Features

| Area | What it does |
|---|---|
| 💸 **Expenses** | Log, edit and delete expenses per department with payment method and an optional receipt (PDF/PNG/JPG, 5 MB cap) |
| 🏛️ **Departments** | Create departments with an icon and colour, assign a manager, activate/deactivate |
| 📊 **Budgets** | Monthly limit per department; managers are blocked from exceeding it |
| 🔔 **Alerts & Queries** | Raise `info / warning / critical` alerts, or address a query to a named person; resolve when done |
| 🗓️ **Month-end reports** | System auto-generates an "End of Month Report" alert per department (spend vs budget) |
| 🙏 **Donations** | Record Cash / UPI / Bank donations, export CSV + PDF, email the report to the temple, forward each donation to an external webhook |
| 📄 **Reports** | Date/department-filtered reports exported as PDF or CSV |
| 🧾 **Activity log** | Every create/update/delete is audit-logged with who and what |
| 👥 **User management** | Approve new sign-ups, change roles, reset passwords, enable/disable or delete users |
| 🔐 **Auth** | Local email + password (Argon2) **or** Clerk (Email/Phone OTP + Google, invite-only org) |
| 📲 **WhatsApp** | Login notifications via Twilio, per-user mode: `all` / `large_only` / `off` |
| ☁️ **Backups** | Receipts zipped and uploaded to Google Drive every 15 days |
| 🎨 **Spiritual UI** | Rotating temple-photo backgrounds on public pages, Chart.js dashboard |

---

## 🏗️ System Architecture

```mermaid
flowchart LR
    subgraph Client["🖥️ Browser"]
        UI["Jinja2 templates<br/>style.css · spiritual.css<br/>main.js · Chart.js"]
    end

    subgraph App["🐍 Flask App (app.py)"]
        direction TB
        MW["before_request guards<br/>auth · pending lockout<br/>TMC read-only · manager scope"]
        RT["Routes<br/>dashboard · expenses · budgets<br/>alerts · reports · donations · users"]
        MW --> RT
    end

    subgraph Services["⚙️ services/"]
        SEC["security.py<br/>Argon2 hashing"]
        CLK["clerk_auth.py<br/>JWT + org check"]
        EML["email_service.py<br/>SMTP"]
        WAS["whatsapp_service.py<br/>Twilio"]
        RPT["report_service.py<br/>donation_report_service.py<br/>ReportLab PDF · CSV"]
        BKP["backup_service.py<br/>Drive upload"]
        SCH["scheduler_service.py<br/>APScheduler"]
    end

    subgraph Data["🗄️ database/db.py"]
        DB[("SQLite (default)<br/>or Supabase PostgreSQL")]
        FS[("Receipts on disk<br/>static/uploads/receipts")]
    end

    subgraph Ext["🌐 External"]
        CLERK["Clerk"]
        GMAIL["Gmail SMTP"]
        TW["Twilio WhatsApp"]
        GD["Google Drive"]
        HOOK["Donations webhook"]
    end

    UI <-->|HTTP| MW
    RT --> SEC & CLK & EML & WAS & RPT
    RT <--> DB
    RT <--> FS
    SCH --> RPT & EML & BKP
    SCH --> DB
    CLK <--> CLERK
    EML --> GMAIL
    WAS --> TW
    BKP --> GD
    FS --> BKP
    RT -->|"POST JSON (3s timeout)"| HOOK
```

---

## 🔄 Project Flow

### Request lifecycle

```mermaid
flowchart TD
    A([User opens a page]) --> B{Signed in?}
    B -- No --> L["/login<br/>Local email+password<br/>or Clerk OTP / Google"]
    L --> B
    B -- Yes --> C{Role?}
    C -- pending --> P["/pending<br/>Awaiting accountant approval"]
    C -- tmc_member --> T{"Write request?"}
    T -- "Yes, not whitelisted" --> X["⛔ Read-only account"]
    T -- "No / alerts / profile" --> R
    C -- manager --> M{"Endpoint in<br/>manager allow-list?"}
    M -- No --> Y["⛔ Scoped to own department"]
    M -- Yes --> R
    C -- accountant --> R["Route handler"]
    R --> V["Validate input<br/>budget cap · file type · amounts"]
    V --> W[("Write to DB")]
    W --> LOG["log_activity()<br/>audit trail"]
    LOG --> OUT([Render page / PDF / CSV])
```

### Typical working day

```mermaid
flowchart LR
    A["Manager logs a bill<br/>+ uploads receipt"] --> B{"Within monthly<br/>budget?"}
    B -- No --> C["❌ Rejected with<br/>remaining balance"]
    B -- Yes --> D["✅ Saved + audit log"]
    D --> E["Dashboard & charts update"]
    E --> F["TMC member reviews,<br/>raises a query to the manager"]
    F --> G["Manager replies,<br/>accountant resolves alert"]
    G --> H["Month-end: system posts<br/>spend-vs-budget alert"]
    H --> I["Export PDF / CSV report"]
```

---

## 🛡️ Roles & Access Control

Roles live in the local `users` table. Clerk only proves *identity and organisation membership*; the app's own role and department assignment stay in the database.

| Capability | 🧾 Accountant | 🏛️ TMC Member | 🧑‍💼 Manager | ⏳ Pending |
|---|:---:|:---:|:---:|:---:|
| Dashboard | ✅ all | ✅ all (read-only) | ✅ own dept | ❌ |
| Add / edit / delete expenses | ✅ | ❌ | ✅ own dept, budget-capped | ❌ |
| Departments (create/edit/delete) | ✅ | ❌ | ❌ | ❌ |
| Budgets (set / delete) | ✅ | ❌ view | view only | ❌ |
| Raise alerts / queries | ✅ | ✅ | ❌ (can view) | ❌ |
| Resolve / delete alerts | ✅ | ❌ | ❌ | ❌ |
| Reports (PDF / CSV) | ✅ | ✅ | ✅ own dept | ❌ |
| Activity log | ✅ | ✅ | ❌ | ❌ |
| Donations: record | ✅ | ❌ | ❌ | ❌ |
| Donations: view / export | ✅ | ✅ | ❌ | ❌ |
| User management | ✅ | ❌ | ❌ | ❌ |

> Enforced server-side by `login_required`, `role_required(...)` and `before_request` guards (`load_logged_in_user`, `enforce_pending_lockout`, `enforce_auditor_readonly`, `enforce_manager_scope`). UI hiding is cosmetic only. Budgets are shown on the merged **Finances** page, and the activity log on the merged **Reports & Audit** page (hidden from managers).

---

## 🙏 Donation Pipeline

Donations are saved **first**; nothing external can ever lose a record.

```mermaid
sequenceDiagram
    autonumber
    actor A as Accountant
    participant F as Flask (/admin/add_donation)
    participant DB as Database
    participant W as External Webhook

    A->>F: Submit donor, amount, method, txn id, notes, DOB
    F->>F: Validate (amount > 0, method ∈ Cash/UPI/Bank, lengths, DOB)
    F->>DB: INSERT donation + COMMIT
    F->>DB: log_activity (non-fatal if it fails)
    F->>W: POST JSON (3 s timeout)
    alt reachable
        W-->>F: 2xx
        F->>DB: webhook_status = sent
    else timeout / error
        F->>DB: webhook_status = failed
    else URL not configured
        F->>DB: webhook_status = not configured
    end
    F-->>A: Flash result (recorded ✔ / not sent ⚠)
```

---

## ⏰ Background Jobs

Enabled with `START_SCHEDULER=true`. One shared APScheduler instance (hence a **single gunicorn worker**).

| Job | Schedule | Runs when |
|---|---|---|
| Month-end spend-vs-budget alerts (idempotent per dept + month) | Daily 02:00 | always |
| Supabase keep-alive ping (emails an URGENT alert if the DB is down) | Daily 03:00 | PostgreSQL only |
| Weekly DB health-check email | Every 7 days | PostgreSQL only |
| Donations CSV + PDF emailed to the temple | Every 7 days | always |
| Receipts zipped → Google Drive | Every 15 days | `GOOGLE_DRIVE_BACKUP_FOLDER_ID` + service-account key set |

---

## 🗃️ Database Schema

```mermaid
erDiagram
    ROLES ||--o{ USERS : "has"
    USERS ||--o{ DEPARTMENTS : "creates / manages"
    DEPARTMENTS ||--o{ EXPENSES : "contains"
    DEPARTMENTS ||--o| BUDGETS : "limited by"
    DEPARTMENTS ||--o{ ALERTS : "about"
    EXPENSES ||--o{ ALERTS : "referenced by"
    USERS ||--o{ EXPENSES : "adds"
    USERS ||--o{ ALERTS : "raises / assigned"
    USERS ||--o{ AUDIT_LOGS : "performs"
    USERS ||--o{ DONATIONS : "records"

    ROLES { int id PK
            text name "accountant, tmc_member, manager, pending" }
    USERS { int id PK
            text email UK
            text password_hash
            text clerk_id UK
            int role_id FK
            text phone_number
            text whatsapp_notify_mode }
    DEPARTMENTS { int id PK
                  text name UK
                  text icon
                  text color
                  int manager_id FK
                  int is_active }
    EXPENSES { int id PK
               int department_id FK
               real amount
               text description
               text date
               text payment_method
               text receipt_path }
    BUDGETS { int id PK
              int department_id FK
              real monthly_limit }
    ALERTS { int id PK
             text type "alert | query"
             text severity "info | warning | critical"
             int is_resolved
             int assigned_to FK }
    AUDIT_LOGS { int id PK
                 text action
                 text entity_type
                 int entity_id
                 text details }
    DONATIONS { int id PK
                text donor_name
                real amount
                text payment_method "Cash | UPI | Bank"
                text transaction_id
                text webhook_status
                date dob }
```

The same schema is created automatically on first run (SQLite or Postgres). `supabase_schema.sql` is provided for setting up Supabase manually.

---

## 📁 Project Structure

```text
iskcon-shirpur-finance-management-system/
└── expense-tracker/
    ├── app.py                      # Flask app: routes, RBAC guards, scheduler bootstrap
    ├── config.py                   # Env-driven settings (DB, Clerk, Drive, email, webhook)
    ├── requirements.txt            # Python dependencies
    ├── Procfile                    # gunicorn entry for production
    ├── supabase_schema.sql         # Idempotent Postgres schema for Supabase
    ├── .env.example                # Copy to .env and fill in
    │
    ├── database/
    │   └── db.py                   # SQLite/Postgres abstraction, pooling, init_db, seed_db, log_activity
    │
    ├── services/
    │   ├── security.py             # Argon2 password hash / verify
    │   ├── clerk_auth.py           # Clerk JWT verification + org-membership guard
    │   ├── email_service.py        # SMTP emails (reset, health, offline alert, reminders)
    │   ├── whatsapp_service.py     # Twilio WhatsApp (DEV console fallback)
    │   ├── report_service.py       # Department PDF reports (ReportLab)
    │   ├── donation_report_service.py  # Donations CSV + PDF + weekly email
    │   ├── backup_service.py       # Zip receipts → Google Drive
    │   └── scheduler_service.py    # APScheduler jobs
    │
    ├── templates/                  # Jinja2 pages
    │   ├── base.html  landing.html  login.html  dashboard.html
    │   ├── finances.html  departments.html  alerts.html  reports_audit.html
    │   ├── admin_donations.html  users.html  profile.html  pending.html
    │   └── forgot_password.html  reset_password.html  clerk_logout.html
    │       terms.html  privacy.html  403.html
    │
    ├── static/
    │   ├── css/   (style.css, spiritual.css)
    │   ├── js/    (main.js, spiritual.js)
    │   ├── fonts/ (DejaVuSans for ₹ in PDFs)
    │   ├── images/  backgrounds/   # logo + rotating temple photos
    │   └── uploads/receipts/       # uploaded bills (git-ignored)
    │
    └── tests/
        └── test_app.py             # pytest suite (auth, RBAC, budgets, alerts, donations, scheduler)
```

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Backend | Flask 3.1, Werkzeug, Gunicorn |
| Database | SQLite (default) · Supabase PostgreSQL via `psycopg2` with connection pooling |
| Auth | Argon2 (`argon2-cffi`) · Clerk (`clerk-backend-api`, PyJWT) |
| Reports | ReportLab (PDF), CSV |
| Jobs | APScheduler |
| Integrations | Twilio (WhatsApp), SMTP (Gmail), Google Drive API |
| Frontend | Jinja2, vanilla JS, Chart.js |
| Testing | pytest, pytest-flask |

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- Git

### Run locally (SQLite, no setup)

1. Clone the repo

```bash
git clone https://github.com/abhinavgupta30/iskcon-shirpur-finance-management-system.git
cd iskcon-shirpur-finance-management-system/expense-tracker
```

2. Create and activate a virtual environment

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
```

3. Install dependencies

```bash
pip install -r requirements.txt
```

4. Create your env file

```bash
cp .env.example .env
```

5. Set a strong secret key in `.env`

```bash
python -c "import secrets; print(secrets.token_hex(32))"
# paste the output as SECRET_KEY=...
```

6. Start the app

```bash
python app.py
```

7. Open **http://127.0.0.1:5001**

On first run the database is created and seeded with demo accounts (accountant, TMC member, manager), sample departments, expenses and alerts. Credentials are defined in `seed_db()` in `database/db.py`. **Change or remove them before going live.**

### Switch to Supabase / PostgreSQL

1. Run `supabase_schema.sql` in the Supabase SQL Editor
2. Set the **direct connection** string (port `5432`, not the 6543 pooler) in `.env`

```env
DATABASE_URL=postgresql://postgres:[PASSWORD]@db.xxxxxxxxxxxx.supabase.co:5432/postgres
```

### Switch to Clerk sign-in

Set all of these in `.env` (leave blank to keep local email/password):

```env
CLERK_SECRET_KEY=
CLERK_PUBLISHABLE_KEY=
CLERK_ORG_ID=org_xxxxxxxx
CLERK_JWT_KEY=          # optional, enables fully offline token checks
```

---

## ⚙️ Configuration

All settings come from `.env` (see `.env.example`). Every integration is **off by default** and activates only when its variables are set.

| Variable | Purpose | Default |
|---|---|---|
| `SECRET_KEY` | Flask session signing | **must be set** |
| `DATABASE_URL` | Postgres/Supabase URL | blank → SQLite |
| `PG_POOL_MINCONN` / `PG_POOL_MAXCONN` | Postgres pool size | `1` / `10` |
| `CLERK_*` | Clerk auth (secret, publishable, org id, JWT key) | blank → local login |
| `GOOGLE_SERVICE_ACCOUNT_FILE` | Service-account JSON path | `credentials.json` |
| `GOOGLE_DRIVE_BACKUP_FOLDER_ID` | Drive folder for receipt backups | blank → backups off |
| `SMTP_SERVER` `SMTP_PORT` `SMTP_USERNAME` `SMTP_PASSWORD` `SENDER_EMAIL` | Outgoing email | Gmail example |
| `WHATSAPP_PROVIDER` `TWILIO_ACCOUNT_SID` `TWILIO_AUTH_TOKEN` `TWILIO_PHONE_NUMBER` | WhatsApp notifications | `DEV` console fallback |
| `DONATIONS_WEBHOOK_URL` | Where each donation is POSTed | blank → skipped |
| `DONATIONS_REPORT_EMAIL` | Weekly donations report recipient | set in config |
| `ADMIN_ALERT_EMAIL` | Receives DB health / offline alerts | set in config |
| `START_SCHEDULER` | Run background jobs | off |
| `DATA_DIR` | Persistent disk for receipts and SQLite in production | project folder |

---

## 🧪 Testing

```bash
cd expense-tracker
pytest -v
```

The suite (~40 tests) forces local auth regardless of your `.env`, so it never depends on real Clerk or Twilio credentials.

---

## ☁️ Deployment

The included `Procfile` runs:

```text
web: gunicorn app:app --workers 1 --threads 4 --timeout 60
```

1. Deploy `expense-tracker/` to any Procfile-based host (Render, Railway, Heroku-style)
2. Add the environment variables above in the host dashboard
3. Mount a persistent disk and set `DATA_DIR` so receipts (and SQLite, if used) survive restarts
4. Set `START_SCHEDULER=true`

> Keep **one worker**. The scheduler lives in-process, so multiple workers would run every job multiple times.

---

## 🔒 Security Notes

- Passwords are hashed with **Argon2**; reset tokens are time-limited
- Role checks are enforced on the server for every route, not only in the UI
- Upload limit of 5 MB; receipt extensions restricted to `pdf, png, jpg, jpeg`
- `.env`, `credentials.json` and uploaded receipts are git-ignored
- **Before production:** set a strong `SECRET_KEY`, change the seeded demo passwords, and set `CLERK_ORG_ID` explicitly

---

## 👤 Author

**Abhinav Gupta** · [@abhinavgupta30](https://github.com/abhinavgupta30)

Built with 🙏 for the service of ISKCON Shirpur.

<div align="center">

**Hare Krishna** 🪷

</div>
