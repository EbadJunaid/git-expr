# SSL Certificate Analytics Dashboard

A full‑stack web application for **exploring and auditing SSL/TLS certificates at scale**.
It transforms the certificates collected by this project's crawlers into interactive dashboards and security reports, providing insights into certificate validity, Certificate Authority (CA) ecosystems, cryptographic algorithms, Subject Alternative Names (SANs), shared public-key reuse, issuer distribution, renewal trends, and overall certificate risk.


- **Backend:** Django 5 providing a REST-style JSON API, communicating directly with **MongoDB** using `pymongo`, with optional **Redis** caching.
- **Frontend:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4, **SWR** for data fetching, and **Recharts** for visualizations.

> [!TIP]
> **First time here?**
>
> Read this document once from top to bottom, then follow **Part 1 → Part 5** in order.
>
> The most common reason the dashboard displays empty pages is that MongoDB has not been populated with certificate data. See **[Preparing the dataset](#-preparing-the-dataset)** before running the application.

---

> [!NOTE]
> **Platform compatibility**
>
> Unless stated otherwise, all commands in this README use Unix-style syntax (`python3` and forward-slash paths `/`).
>
> **Windows users:**
> - Replace `python3` with `python`.
> - You may use backslashes (`\`) instead of forward slashes (`/`) in file paths if you prefer.


## Table of contents

1. [What the dashboard does](#-what-the-dashboard-does)
2. [Architecture at a glance](#-architecture-at-a-glance)
3. [Tech stack](#-tech-stack)
4. [Prerequisites](#-prerequisites)
5. [Preparing the dataset](#-preparing-the-dataset)
6. [Running the dashboard](#-running-the-dashboard)
7. [Project structure](#-project-structure)
8. [Troubleshooting](#-troubleshooting)

---

## 🔎 What the dashboard does

The dashboard presents parsed SSL/TLS certificates through **11 analytics pages**, each focusing on a different aspect of certificate analysis and security.

All routes below are relative to `/dashboard`.

| Route | What it shows |
|---|---|
| `overview` | Home page: total / active / expiring‑soon / expired certificate counts, plus a searchable, paginated certificate table. |
| `ca‑analytics` | Certificate Authority market share, top CAs, issuer × validation‑level matrix, self‑signed analysis. |
| `ca‑ranking` | Ranks Certificate Authorities based on certificate quality and security characteristics. See [`ca-ranking.md`](../formula/ca-ranking.md) |
| `validity‑analytics` | Validity period analysis — average lifetime, expiring in 30/90 days, compliance with the 398‑day rule, issuance timeline. |
| `signature‑hash` | Signature algorithms, hash compliance %, weak‑hash alerts, key‑size distribution, adoption trends. |
| `san‑analytics` | Subject Alternative Names — SANs per certificate, wildcard vs standard, top TLDs, multi‑domain ("blast radius") certs. |
| `shared‑keys` | Public‑key **reuse** detection (a security risk): groups of certificates sharing the same key. Drill into a group via `shared‑keys/[publicKeyHash]`. |
| `vulnerabilities` | A **ranked risk view** scoring each certificate. See [`vulnerabilities.md`](./vulnerabilities.md) for the exact scoring formula. |
| `active‑vs‑expired` | Detailed breakdown of active, expiring, and expired certificates |
| `issuer‑countries` | Distribution of certificates by country — e.g. how many belong to Pakistan (`.pk`), based on our dataset. |
| `cas‑vs‑domains` | Shows how many certificates each Certificate Authority has issued — e.g. how many were issued by Let's Encrypt |

Additional detail pages: 

- `/certificate/[id]` — full inspection of a single certificate (validity, issuer, subject, fingerprints, SANs, key usage, ZLint results etc).
- `/dashboard/shared-keys/[publicKeyHash]` — every certificate that shares one public key.

**Key features**

- **Scope switcher** — the same physical database can be sliced by country (Global, Pakistan, India, …). The frontend appends a `?scope=` parameter to every request; the backend filters accordingly. (Switcher UI in the header; backed by `Scopes.json`.)
- **Dark / light theme**, global search and advanced filtering. 

---

## 🏗 Architecture at a glance

```
                 Browser (you)
                       │  http://localhost:3000
                       ▼
        ┌──────────────────────────────┐
        │   Frontend — Next.js 16       │   React 19 + Tailwind v4 + SWR + Recharts
        │   src/app/dashboard/*         │
        └──────────────┬───────────────┘
                       │  fetch JSON  →  http://localhost:8000/api/...?scope=all
                       ▼
        ┌──────────────────────────────┐
        │   Backend — Django 5          │   views → controllers → db_queries
        │   certificates/ app           │   (+ optional Redis cache)
        └──────────────┬───────────────┘
                       │  pymongo
                       ▼
        ┌──────────────────────────────┐
        │   MongoDB  (localhost:27017)  │
        │   • hugging-face-700k         │  ← raw parsed certificates
        │   • hugging-face-700k-results │  ← PRE-COMPUTED analytics (for speed)
        └──────────────────────────────┘
```

**Backend request flow (per analytics module):** Each analytics module under `certificates/` (`ca_analytics`, `overview`, `san_analytics`, `shared_keys`, `signature_hash`, `validity_analysis`, `shared_apis`) follows the same architecture:

```
urls.py  →  views.py  →  controllers.py  →  db_queries.py  →  MongoDB
                              │
                              └─ checks Redis cache first (if available)
```

To keep pages fast, heavy aggregations are **pre‑computed** into the `*-results` database by the
scripts in `pre-compute-scripts/`. The live API mostly reads those
pre‑computed collections instead of scanning millions of documents on every click.

---

## 🧰 Tech stack

| Layer | Technology | Version (from code) |
|---|---|---|
| Frontend framework | Next.js (App Router) | 16.1.1 |
| UI library | React / React DOM | 19.2.3 |
| Language | TypeScript | 5.x |
| Styling | Tailwind CSS | v4 |
| Data fetching | SWR | 2.3.x |
| Charts | Recharts | 3.6.x |
| Icons | @heroicons/react | 2.2.x |
| Backend framework | Django | 5.x (built on 5.2) |
| Mongo driver | pymongo | 4.6+ |
| CORS | django‑cors‑headers | 4.3+ |
| Cache (optional) | redis | 5.0+ |
| Database | MongoDB | 27017 (local) |






## ✅ Prerequisites

Install the following software before setting up the dataset or running the dashboard.

### 1. Python 3.11+

Check whether Python is already installed:

```bash
python3 --version
```

Missing? Download it from <https://www.python.org/>.

Create and activate a virtual environment (recommended):

```bash
python3 -m venv venv
source venv/bin/activate
```

> [!NOTE]
> **Windows users:**
>
> Replace `python3` with `python`, then activate the environment using:
>
> ```powershell
> venv\Scripts\activate
> ```
>
> If you already use `pyenv`, `pyenv-virtualenv`, or Conda, you can continue using your preferred environment manager.
---

### 2. Node.js + npm

Check whether npm is installed:

```bash
npm --version
```

Missing? Install Node.js (npm is included) from <https://nodejs.org/>.

> [!NOTE]
> The project should also work with `bun`, but only the `npm` workflow has been tested.

---

### 3. MongoDB (required)

The dashboard requires:

- MongoDB Community Server
- MongoDB Database Tools (`mongorestore`, `mongodump`)

Check your installation:

```bash
mongod --version
mongorestore --version
```

If anything is missing, install:

- MongoDB Community Server: <https://www.mongodb.com/try/download/community>
- MongoDB Database Tools: <https://www.mongodb.com/try/download/database-tools>

(Optional)

- MongoDB Compass: <https://www.mongodb.com/try/download/compass>

Make sure the MongoDB server is running before continuing.

---

### 4. Redis (optional)

Redis is used only for caching. The dashboard works perfectly without it.

Check whether Redis is running:

```bash
redis-cli ping
```

If Redis is available, the command should return:

```text
PONG
```

---

## 📦 Preparing the dataset

This repository contains the dashboard source code, **not** the certificate dataset.

Before running the dashboard, MongoDB must contain **both** of the following databases:

| Database | Purpose |
|----------|---------|
| `hugging-face-700k` | Raw parsed SSL/TLS certificates (`certificates` collection) |
| `hugging-face-700k-results` | Pre-computed analytics used by the dashboard for fast queries |

If these databases do not exist, the dashboard will still start, but the pages will be empty because there is no data to display.

The dashboard reads its MongoDB connection settings and database names from the shared [`project-config.json`](../project-config.json) file.

> [!TIP]
> Want to see what the dataset looks like before downloading or crawling it?
> Check [`data-sample.json`](../data-sample.json), which contains a real parsed certificate exactly as it is stored in MongoDB.

There are three ways to prepare the dataset.

### Method 1 — Ready-made dataset (recommended)

This is the fastest option.

Download both dataset files from the [Hugging Face](https://huggingface.co/datasets/EbadJunaid/hugging-face-700k-ssl-certificates-data/tree/main).

After downloading, you should have:

- `hugging-face-700k.archive.gz`
- `hugging-face-700k-results.archive.gz`

Restore both databases:

```bash
cd path/to/downloaded/files

mongorestore --archive="hugging-face-700k.archive.gz" --gzip
mongorestore --archive="hugging-face-700k-results.archive.gz" --gzip
```

The first command restores the raw certificate dataset.

The second restores the pre-computed analytics used by the dashboard.

> [!NOTE]
> The pre-computed analytics remain accurate over time. The only section that gradually becomes outdated is the monthly issuance timeline. As new months begin, the timeline will not include the newest month until you regenerate the analytics by running `run-generic.py`. All other analytics remain accurate.

---

### Method 2 — Build fresh analytics

Use this option if you want the pre-computed results to always be up to date. Instead of using our ready-made analytics database, you can generate it yourself from the main certificate database.

Download only the [main dataset](https://huggingface.co/datasets/EbadJunaid/hugging-face-700k-ssl-certificates-data/blob/main/hugging-face-700k-results.archive.gz) from Hugging Face and restore it:

```bash
cd path/to/downloaded/files

mongorestore --archive="hugging-face-700k.archive.gz" --gzip
```

Then generate the analytics database:

```bash
cd dashboard/backend/pre-compute-scripts

python3 run-generic.py
```

The script rebuilds the entire `hugging-face-700k-results` database from the raw certificates.

Generating the analytics database can take anywhere from several minutes to over an hour, depending on your system's available memory and disk performance.

---

### Method 3 — Build everything yourself

This option is intended for users who want to collect their own SSL/TLS certificates instead of using the published dataset.

Run the [Domain-based crawler](../README-main-final.md#1-domainbased-crawler) described in the main project README to build your own certificate dataset.


Once the crawl has finished, generate the dashboard analytics database:

```bash
cd dashboard/backend/pre-compute-scripts

python3 run-generic.py
```

> [!NOTE]
> Database names, MongoDB connection settings, and the default input CSV are shared across the project through [`project-config.json`](../project-config.json). Updating this file automatically keeps the crawler, dashboard backend, pre-compute scripts, and CT logs renewal pipeline in sync.





## 🚀 Running the dashboard

Before continuing, make sure you have already completed the following sections:

- ✅ [Prerequisites](#-prerequisites)
- ✅ [Preparing the dataset](#-preparing-the-dataset)

At this point you should have:

- A Python virtual environment created and activated.
- Node.js and npm installed.
- MongoDB running.
- The certificate database restored (or collected using the crawler).
- The analytics results database available.

The dashboard consists of two independent applications:

- **Backend** — Django API (`localhost:8000`)
- **Frontend** — Next.js application (`localhost:3000`)

Run each one in a separate terminal and keep both running.

---

### 1. Start the backend

From the project root:

```bash
cd dashboard/backend
```

Install the Python dependencies (only the first time):

```bash
pip install -r requirements.txt
```

If you are using custom database names or a MongoDB URI other than `localhost:27017`, update the shared configuration in `project-config.json`:

```text
project-config.json
```

Then initialize Django's internal SQLite database (only the first time):

```bash
python3 manage.py migrate
```

Start the backend server:

```bash
python3 manage.py runserver
```

The backend will be available at:

```text
http://localhost:8000
```

---

### 2. Start the frontend

Open a **new terminal**.

Navigate to the frontend:

```bash
cd dashboard/frontend
```

Install the JavaScript dependencies (only the first time):

```bash
npm install
```

Start the development server:

```bash
npm run dev
```

The frontend will be available at:

```text
http://localhost:3000
```

---

### 3. Open the dashboard

Open your browser and visit:

```text
http://localhost:3000
```

You should now see the dashboard.

If everything is configured correctly, the data flow is:

```text
MongoDB
    │
    ▼
Django Backend (:8000)
    │
    ▼
Next.js Frontend (:3000)
```


> [!TIP]
> If the dashboard starts successfully but displays empty pages, the most common causes are:
>
> - the dataset has not been prepared correctly, or
> - the database name or MongoDB connection configured in `project-config.json` does not match your MongoDB instance.
>
> See **[Preparing the dataset](#-preparing-the-dataset)** if you're unsure.


## 📁 Project structure

```text
dashboard/
├── backend/
│   ├── certificates/            # Django application and analytics modules
│   ├── pre-compute-scripts/     # Builds the analytics results database
│   ├── ssl_dashboard/           # Django project configuration
│   ├── country-domain-extractors/
│   ├── manage.py
│   └── requirements.txt
│
├── frontend/
│   ├── src/
│   │   ├── app/                 # Next.js routes
│   │   ├── components/          # Shared UI components
│   │   ├── context/             # React contexts
│   │   ├── hooks/               # Custom React hooks
│   │   ├── providers/           # Global providers
│   │   └── services/            # Backend API client
│   └── package.json
│
├── data-sample.json             # Example parsed certificate
├── vulnerabilities.md           # Risk scoring documentation
└── README.md
```

For a complete overview of the entire project (including the crawlers and CT renewal pipeline), see the **main project README**.


## 🩺 Troubleshooting

| Problem | Likely cause & solution |
|---------|------------------------|
| Dashboard shows **no certificate data** | The main certificate database is missing, empty, or its name in `project-config.json` does not match MongoDB. Verify the database name and complete **Preparing the dataset**. |
| Dashboard keeps showing **Loading...** on analytics pages | The analytics results database has not been generated, is empty, or its name in `project-config.json` does not match MongoDB. Generate the analytics database (see **Method 1** or **Method 2** in **Preparing the dataset**) and verify its configured name. |
| Backend cannot connect to MongoDB | Make sure MongoDB is running and that the MongoDB URI in `project-config.json` is correct. |
| Port already in use | **Backend:** run Django on another port, for example `python3 manage.py runserver 8001`, then set `NEXT_PUBLIC_API_URL=http://localhost:8001/api` in `frontend/.env.local`. **Frontend:** run `npm run dev -- -p 3001` and add `http://localhost:3001` to `CORS_ALLOWED_ORIGINS` in `backend/ssl_dashboard/settings.py`. Restart both servers after making the changes. |
| `python3` or `npm` command not found | Make sure Python and Node.js are installed and available in your system's `PATH`. |