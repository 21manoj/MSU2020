# MSU Vision 2020 — MVP (Django)

Runnable minimum implementation aligned with `msu_vision_2020_architecture_v1.2.md`: stakeholders (profiles + 8 roles), needs lifecycle, donor matching, projects, milestones with Gantt-style timeline, fund pools, contributions (including event-linked pledges), expenses, events with media, governance thresholds, persona-aware dashboard, django-allauth (username/email + optional Google).

`load_demo_data` seeds a **boys hostel remodeling** storyline (monthly milestones), **donors**, **July 2026 fundraising gala** (schedule + placeholder photos), and **governance queue** items — see command output for test usernames (`hod_hostel`, `lead_hostel`, `donor_anita`, `gov_meera`, etc.).

## Quick start

```bash
cd msu_vision_2020
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # optional
python manage.py migrate
python manage.py load_demo_data
python manage.py runserver
```

Open http://127.0.0.1:8000/accounts/login/ — use **`demo`** / **`demo`**, or click **Sign in as demo** (DEBUG only). Other demo users use password `demo123` (see `load_demo_data` output).

## Production notes

- Target stack in the architecture doc: Python 3.12, Django 5.1, PostgreSQL, Redis/Celery, R2, Render. This repo uses Django 4.2 + SQLite by default for local MVP on older Python.
- Set `DATABASE_URL`, `SECRET_KEY`, `ALLOWED_HOSTS`, and Google OAuth vars in `.env` for deployment.

## Layout

Matches §7 structure in simplified form: `apps/core`, `stakeholders`, `needs`, `projects`, `funding`, `events`, `dashboard`. Email ingestion, audit history, and PDF export are not implemented in this MVP slice.
