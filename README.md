# MSU Vision 2020

Django app for alumni–foundation coordination: **needs**, **projects** (milestones / timeline), **funding** (pools, contributions, expenses), **events**, **governance** thresholds, and persona-aware dashboards.

Product/architecture details may live in `msu_vision_2020_architecture_v1.2.md` if you add that file to this repository.

---

## Prerequisites

- **Python 3.9+** (Django 4.2 locally; **3.11** matches the production `Dockerfile`)
- **Git** and **pip**; virtualenv recommended

Optional: **Docker**; **AWS CLI + EB CLI** only for deploy.

---

## Local setup (co-developers)

```bash
git clone git@github.com:21manoj/MSU2020.git
cd MSU2020

python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env
# Edit .env — set SECRET_KEY for any shared/staging use

python manage.py migrate
python manage.py load_demo_data
python manage.py runserver
```

Open **http://127.0.0.1:8000/accounts/login/**

- **`demo`** / **`demo`**, or **Sign in as demo** when `DEBUG=True`
- Other seeded users: **`demo123`** (see `load_demo_data` output: `hod_hostel`, `donor_anita`, `gov_meera`, …)

---

## Environment variables

| Variable | Purpose |
|----------|--------|
| `DEBUG` | `True` locally; `False` in production |
| `SECRET_KEY` | **Required** in production |
| `ALLOWED_HOSTS` | Comma-separated hosts |
| `CSRF_TRUSTED_ORIGINS` | HTTPS origins if applicable |
| `DATABASE_URL` | Optional; default SQLite (`db.sqlite3`) |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Optional Google login |
| `GOVERNANCE_THRESHOLD_*_USD` | Governance routing thresholds |

**Never commit `.env`** (see `.gitignore`).

---

## Docker (optional)

```bash
docker build -t msu-vision-2020 .
docker run --rm -p 8000:8000 --env-file .env msu-vision-2020
```

---

## Django apps

| App | Role |
|-----|------|
| `apps.core` | Shared utilities |
| `apps.stakeholders` | Profiles, personas, registration |
| `apps.needs` | Needs lifecycle |
| `apps.projects` | Projects, milestones, timeline |
| `apps.funding` | Pools, contributions, expenses |
| `apps.events` | Events |
| `apps.dashboard` | Home, rollup, governance queue |

---

## Tests

```bash
source .venv/bin/activate
python manage.py test
```

---

## Deploy (Elastic Beanstalk)

Requires **AWS credentials** and EB config (`.elasticbeanstalk/` — not committed by default). From this directory:

```bash
eb deploy
```

GitHub write access alone does **not** deploy; grant AWS or CI separately.

---

## Collaboration

- **GitHub:** Settings → Collaborators for push/PRs.
- **AWS:** IAM or CI for `eb deploy`.

## License

Add a `LICENSE` file if needed.
