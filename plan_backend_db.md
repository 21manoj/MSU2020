# MSU VISION 2020 — Backend & Database Implementation Plan

> Every item from all 4 planning documents is covered. Nothing is skipped.

---

## Phase 1 — New & Modified Django Models

### 1.1 Contribution Model — Store Exchange Rate at Contribution Time (C-02)
- **File**: `apps/funding/models.py`
- **Current**: `amount_usd` computed on save via static `_to_usd()` using `settings.MVP_EXCHANGE_RATES_TO_USD`
- **Add fields**:
  - `exchange_rate_used = DecimalField(max_digits=12, decimal_places=6, null=True)` — rate snapshot
  - `exchange_rate_date = DateField(null=True, blank=True)` — date rate was fetched
- **Modify `save()`**: store the rate used alongside the USD conversion
- **Migration**: `0002_contribution_exchange_rate_fields`

### 1.2 FundPool Model — Add Computed Balance Properties (H-02)
- **File**: `apps/funding/models.py`
- **Add methods** to `FundPool`:
  - `total_collected_usd()` — sum of `contributions.filter(status__in=[received, allocated, utilized]).aggregate(Sum('amount_usd'))`
  - `total_allocated_usd()` — sum of `expenses.filter(status__in=[approved, disbursed]).aggregate(Sum('amount_usd'))`
  - `total_remaining_usd()` — `collected - allocated`
  - `utilization_percent()` — `(allocated / collected) * 100`

### 1.3 UserProfile Model — Expand Profile Fields (H-04)
- **File**: `apps/stakeholders/models.py`
- **Add fields**:
  - `batch_year = PositiveSmallIntegerField(null=True, blank=True)`
  - `department = CharField(max_length=255, blank=True)`
  - `phone = CharField(max_length=32, blank=True)`
  - `linkedin_url = URLField(blank=True)`
  - `bio = TextField(blank=True)`
  - `photo = ImageField(upload_to='profile_photos/%Y/', blank=True, null=True)`
- **Migration**: `0002_userprofile_expanded_fields`

### 1.4 UserRoleRequest Model [NEW] (Doc 3 §6)
- **File**: `apps/stakeholders/models.py`
- **Fields**:
  - `user = ForeignKey(AUTH_USER_MODEL, on_delete=CASCADE, related_name='role_requests')`
  - `requested_persona = CharField(max_length=32, choices=StakeholderType.choices)`
  - `reason = TextField()`
  - `status = CharField(choices=[pending, approved, rejected], default=pending)`
  - `reviewed_by = ForeignKey(AUTH_USER_MODEL, null=True, blank=True, related_name='role_reviews')`
  - `review_notes = TextField(blank=True)`
  - `reviewed_at = DateTimeField(null=True, blank=True)`
- **Constraints**: Unique on `(user, requested_persona, status='pending')` — prevent duplicate pending requests
- **Migration**: `0003_userrolerequest`

### 1.5 UserStakeholderPersona Model — Add Missing Fields (Doc 3 §5a)
- **File**: `apps/stakeholders/models.py`
- **Current**: has `user`, `persona_type` — missing `assigned_by`, `assigned_at`, `is_active`
- **Add fields**:
  - `assigned_by = ForeignKey(AUTH_USER_MODEL, null=True, blank=True, related_name='persona_assignments')`
  - `assigned_at = DateTimeField(auto_now_add=True)`
  - `is_active = BooleanField(default=True)`
- **Update** `persona_codes()` in `UserProfile` to filter `is_active=True`
- **Migration**: `0004_userstakeholderpersona_extended`

### 1.6 EventMilestone Model [NEW] (Doc 1 §4a, Doc 2 §3)
- **File**: `apps/events/models.py`
- **Fields**:
  - `event = ForeignKey(Event, on_delete=CASCADE, related_name='milestones')`
  - `title = CharField(max_length=255)`
  - `owner = ForeignKey(AUTH_USER_MODEL, on_delete=SET_NULL, null=True)`
  - `due_date = DateField()`
  - `completed_date = DateField(null=True, blank=True)`
  - `proof = FileField(upload_to='event_milestone_proof/%Y/', blank=True, null=True)`
  - `proof_original_filename = CharField(max_length=255, blank=True)`
  - `completed = BooleanField(default=False)`
  - `sequence = PositiveIntegerField(default=0)`
  - `notes = TextField(blank=True)`
- **Migration**: `0002_eventmilestone`

### 1.7 Event Model — Add Governance Workflow Fields (Doc 1 §4a)
- **File**: `apps/events/models.py`
- **Current Status choices**: `DRAFT, PUBLISHED, REGISTRATION_OPEN, ONGOING, COMPLETED, ARCHIVED, CANCELLED`
- **Add status**: `GOVERNANCE_APPROVED = "governance_approved", "Governance Approved"` (between Draft and Published)
- **Add fields**:
  - `fund_pool = ForeignKey(FundPool, null=True, blank=True, related_name='events')` — linked pool for fundraising events
  - `target_audience = TextField(blank=True)` — who the event targets
  - `is_fundraising = BooleanField(default=False)` — flag for governance workflow
  - `originated_by = ForeignKey(AUTH_USER_MODEL, null=True, blank=True, related_name='events_originated')` — governance originator

### 1.8 Program Model [NEW] (Doc 1 §4b, Doc 2 §3)
- **File**: NEW `apps/programs/models.py` (new Django app `apps.programs`)
- **Fields**:
  - `title = CharField(max_length=255)`
  - `description = TextField(blank=True)`
  - `status = CharField(choices=[draft, active, completed, archived], default=draft)`
  - `fund_pool = ForeignKey(FundPool, null=True, blank=True)`
  - `originated_by = ForeignKey(AUTH_USER_MODEL, on_delete=PROTECT, related_name='programs_originated')`
  - `budget = DecimalField(max_digits=14, decimal_places=2, default=0)`
  - `budget_currency = CharField(max_length=8, default='INR')`
  - `start_date = DateField(null=True, blank=True)`
  - `end_date = DateField(null=True, blank=True)`
  - `owners = ManyToManyField(AUTH_USER_MODEL, related_name='owned_programs')`

### 1.9 ProgramMilestone Model [NEW] (Doc 1 §4b, Doc 2 §3)
- **File**: `apps/programs/models.py`
- **Fields**:
  - `program = ForeignKey(Program, on_delete=CASCADE, related_name='milestones')`
  - `phase = CharField(max_length=255)` — phase grouping (e.g. "Cohort Selection", "Mentorship")
  - `title = CharField(max_length=255)`
  - `owner = ForeignKey(AUTH_USER_MODEL, on_delete=SET_NULL, null=True)`
  - `due_date = DateField()`
  - `completed_date = DateField(null=True, blank=True)`
  - `tranche_percent = PositiveSmallIntegerField(default=0)`
  - `proof = FileField(upload_to='program_milestone_proof/%Y/', blank=True, null=True)`
  - `completed = BooleanField(default=False)`
  - `sequence = PositiveIntegerField(default=0)`
  - `tranche_governance_status = CharField(choices=same as Milestone.TrancheGovernance)`
  - `notes = TextField(blank=True)`

### 1.10 Project Model — Add Program Link (Doc 1 §4b)
- **File**: `apps/projects/models.py`
- **Add field**: `program = ForeignKey('programs.Program', null=True, blank=True, on_delete=SET_NULL, related_name='projects')`
- **Purpose**: incubator projects link to parent program for rollup

### 1.11 UploadAuditLog Model [NEW] (Doc 1 §5b, Doc 2 §3)
- **File**: `apps/core/models.py`
- **Fields**:
  - `uploader = ForeignKey(AUTH_USER_MODEL, on_delete=SET_NULL, null=True)`
  - `filename = CharField(max_length=512)`
  - `file_size = PositiveIntegerField()`
  - `declared_mime = CharField(max_length=255, blank=True)`
  - `actual_mime = CharField(max_length=255, blank=True)`
  - `scan_result = CharField(choices=[clean, flagged, skipped, error], default=clean)`
  - `scan_detail = TextField(blank=True)`
  - `upload_type = CharField(choices=[csv, document, photo, proof])`
  - `action_taken = CharField(choices=[accepted, rejected, quarantined])`
  - `timestamp = DateTimeField(auto_now_add=True)`

### 1.12 New Persona Codes (Doc 3 §4)
- **File**: `apps/stakeholders/models.py` → `UserProfile.StakeholderType`
- **Add choices**:
  - `STUDENT_BENEFICIARY = "student_beneficiary", "Student Beneficiary"`
  - `CSR_CORPORATE_DONOR = "csr_corporate_donor", "CSR / Corporate Donor"`
  - `PROGRAM_MANAGER = "program_manager", "Program Manager"`

---

## Phase 2 — New Django App: `apps.programs`

### 2.1 App Setup
- Create `apps/programs/` with `__init__.py`, `apps.py`, `models.py`, `views.py`, `urls.py`, `forms.py`, `admin.py`, `visibility.py`
- Register in `config/settings.py` → `INSTALLED_APPS`: `"apps.programs.apps.ProgramsConfig"`
- Add URL: `path("programs/", include("apps.programs.urls"))` in `config/urls.py`

### 2.2 Views
- `ProgramListView` — all personas can view (read-only for most)
- `ProgramDetailView` — with milestones, progress, tranche status
- `ProgramCreateView` — restricted to Governance + Foundation Admin
- `ProgramUpdateView` — restricted to Governance + Foundation Admin
- `ProgramMilestoneCreateView` — restricted to Governance + Foundation Admin
- `ProgramMilestoneUpdateView` — restricted to Governance + Foundation Admin
- `program_milestone_tranche_release` — same pattern as project milestone tranche release

### 2.3 Visibility & Permissions
- `can_manage_program(user)` — Governance + Foundation Admin only
- `filter_programs_for_user(qs, user)` — Full for Gov/Admin, read-only for others
- Program Manager persona: full edit on assigned programs only

### 2.4 Forms
- `ProgramForm` — title, description, fund_pool, budget, dates, owners
- `ProgramMilestoneForm` — phase, title, owner, due_date, tranche_percent, proof

---

## Phase 3 — Backend Views & Logic Changes

### 3.1 C-01: Funding Table — Filters & Pagination
- **File**: `apps/funding/filters.py` (already created in earlier session)
- **File**: `apps/funding/views.py` — `ContributionListView` already updated to `FilterView`
- **Verify**: `ContributionFilter` has search, status, fund_pool filters + `OrderingFilter`
- **Add**: `ExpenseFilter` with same pattern for expense list

### 3.2 C-02: Currency Conversion — Dashboard Context
- **File**: `apps/dashboard/views.py` → `HomeView.get_context_data()`
- **Add to context**: `exchange_rates` dict from settings, `exchange_rate_date` (current date for MVP)
- **Add to context**: `currency_toggle` session flag (INR vs USD)
- **Add view**: `toggle_currency(request)` — POST view to flip session flag

### 3.3 C-03: CSV Verification Pipeline (Doc 1 §5a)
- **File**: `apps/stakeholders/persona_utils.py`
- **Current**: `parse_profile_upload_csv()` does basic parsing; `apply_profile_bulk_rows()` processes immediately
- **Modify `parse_profile_upload_csv()`**:
  - Add encoding check (reject non-UTF-8) — already partially done
  - Add row limit check (500 max)
  - Add email format validation per row
  - Add persona code validation per row
  - Add jurisdiction value validation per row
- **Add `verify_csv_rows(rows)`**:
  - Check duplicates against DB (username, email)
  - Return `{valid_rows: [...], flagged_rows: [...], errors: [...]}`
- **Modify upload flow**: two-step — step 1: verify + preview; step 2: confirm + commit
- **File**: `apps/dashboard/views.py`
  - Add `governance_csv_preview(request)` — POST: parse CSV, verify, store in session, return preview
  - Add `governance_csv_confirm(request)` — POST: read from session, apply_profile_bulk_rows, clear session

### 3.4 H-01: Rollup Alerts — Auto-Flag Stalled Projects
- **File**: `apps/dashboard/views.py` → `ProjectRollupView.get_context_data()`
- **Current**: shows `overdue_count` per project
- **Add logic**: for each project with `status=in_progress`, check if ALL milestones have 0% progress AND `created_at` is 14+ days ago → flag as "stalled"
- **Add to row dict**: `is_stalled = bool`, `stalled_days = int`

### 3.5 H-02: Fund Pools — Balance Aggregation
- **File**: `apps/funding/views.py` → `FundPoolListView`
- **Override `get_context_data()`**: annotate each pool with:
  - `total_collected` (sum contributions received/allocated/utilized)
  - `total_allocated` (sum expenses approved/disbursed)
  - `total_remaining`
  - `utilization_pct`

### 3.6 H-03: Project Creation UX — Backend
- **File**: `apps/needs/views.py` → `NeedDetailView.get_context_data()`
- Ensure `can_create_project` context var is correct (already exists)
- No backend change needed — this is primarily a frontend task

### 3.7 H-04: Profile — Expanded Form
- **File**: `apps/stakeholders/forms.py` → `UserProfileForm`
- **Current fields**: `["organization", "stakeholder_type", "jurisdiction", "email_opt_in"]`
- **Add fields**: `["batch_year", "department", "phone", "linkedin_url", "bio", "photo"]`
- Add `first_name`, `last_name` from `User` model (separate form or combined)
- **Add**: recent activity query in `ProfileUpdateView.get_context_data()` — last 10 actions (needs created, milestones updated, contributions made)

### 3.8 M-02: Email Notifications (Doc 1 §M-02)
- **File**: NEW `apps/core/notifications.py`
- **Functions**:
  - `notify_need_status_change(need, old_status, new_status, actor)`
  - `notify_project_status_change(project, old_status, new_status, actor)`
  - `notify_expense_status_change(expense, old_status, new_status, actor)`
  - `notify_milestone_completed(milestone, actor)`
  - `notify_role_request(role_request)` — to Governance + Admin
  - `notify_role_decision(role_request)` — to requester
  - `notify_upload_flagged(audit_log)` — to Foundation Admin
- **File**: NEW `apps/core/signals.py` or per-app signals
- Connect Django signals to send emails on status changes
- **Settings**: Add `EMAIL_BACKEND` SES config; for dev use console backend (already set)

### 3.9 M-03: Audit Trail (Doc 1 §M-03)
- **Add to requirements.txt**: `django-simple-history>=3.5`
- **Add to settings**: `INSTALLED_APPS += ['simple_history']`, `MIDDLEWARE += ['simple_history.middleware.HistoryRequestMiddleware']`
- **Add `HistoricalRecords()`** to models: Need, Project, Milestone, Contribution, Expense, FundPool, Event, UserProfile, UserStakeholderPersona
- **Add view**: `AuditTrailView` — Foundation Admin can view history; CSV export endpoint

### 3.10 M-04: Google OAuth (Doc 1 §M-04)
- **File**: `config/settings.py`
- **Current**: `django-allauth` installed with Google provider configured but credentials empty
- **Action**: Document the setup steps; map Google email to existing user on first login
- **Add post-login signal**: if Google email matches existing user email, link accounts; assign persona via Governance

### 3.11 File Upload Security Pipeline (Doc 1 §5b)
- **File**: NEW `apps/core/upload_security.py`
- **Add to requirements.txt**: `python-magic>=0.4`, `pyclamd>=0.4` (optional)
- **Functions**:
  - `check_mime_type(file_obj, declared_extension)` — uses `python-magic`
  - `check_file_size(file_obj, max_bytes)` — 10MB docs, 5MB photos
  - `scan_for_malware(file_obj)` — ClamAV via `pyclamd` (graceful fallback if ClamAV unavailable)
  - `process_upload(file_obj, uploader, upload_type)` — orchestrator: MIME → size → scan → log → accept/reject/quarantine
- **Integrate** into all upload points:
  - `apps/needs/attachments.py` → `save_need_attachments()`
  - `apps/projects/attachments.py` → `save_project_attachments()`
  - `apps/projects/views.py` → milestone proof upload
  - `apps/events/views.py` → event media upload
  - `apps/programs/views.py` → program milestone proof upload
  - `apps/dashboard/views.py` → CSV upload

### 3.12 Upload Audit Log View [NEW] (Doc 4 §5)
- **File**: `apps/core/views.py`
- **Add `UploadAuditLogListView`**: Foundation Admin full, Governance + Auditor read-only, others 403
- **URL**: `path("audit/uploads/", ...)` in dashboard or core URLs

### 3.13 Role Request Views [NEW] (Doc 3 §6, Doc 4 §5)
- **File**: `apps/stakeholders/views.py`
- **Add views**:
  - `RoleRequestCreateView` — any authenticated user can submit from Profile
  - `RoleRequestListView` — user sees own requests; Governance/Admin sees all pending
  - `role_request_approve(request, pk)` — Governance/Admin action
  - `role_request_reject(request, pk)` — Governance/Admin action with reason
- **Business rules**:
  - Foundation Admin requests → only existing Foundation Admin can approve
  - On approval → create `UserStakeholderPersona` row with `assigned_by`, `is_active=True`
  - On reject → record reason, notify user

### 3.14 Event Milestone Views [NEW] (Doc 4 §5)
- **File**: `apps/events/views.py`
- **Add views**:
  - `EventMilestoneCreateView` — Governance + Foundation Admin
  - `EventMilestoneUpdateView` — Governance + Foundation Admin
  - `event_milestone_complete(request, pk)` — mark complete with proof
- **URL patterns** in `apps/events/urls.py`

### 3.15 Event Create/Edit Views [NEW] (Doc 4 §3)
- **File**: `apps/events/views.py`
- **Current**: only list + detail views exist; no create/edit
- **Add**:
  - `EventCreateView` — Governance + Foundation Admin (fundraising events require governance workflow)
  - `EventUpdateView` — Governance + Foundation Admin
  - `event_transition(request, pk)` — status transitions (Draft → Governance Approved → Published → Completed)
- **Forms**: `EventForm` in new `apps/events/forms.py`

### 3.16 Rollup — Add Programs Section (Doc 1 §4b)
- **File**: `apps/dashboard/views.py` → `ProjectRollupView`
- **Add to context**: `program_rows` — all programs with phase progress and pool utilization
- **Add to context**: incubator projects grouped under parent program

### 3.17 Permissions Updates (Doc 4 — Full Matrix)
- **File**: `apps/core/permissions.py`
- **Update `roles_required()`** decorator → return HTTP 403 `PermissionDenied` instead of redirect to home
- **Add `persona_required()` decorator** — alias or replacement matching Doc 4 §6 pattern
- **Update all views** to enforce the full access matrix from Doc 4 §3:
  - Needs: HOD=Own, Donor=No, Volunteer=No, Finance=No
  - Projects: Donor=Own, Volunteer=Own
  - Funding: Donor=Own, ProjectLead=Limited
  - Pools: ProjectLead=Limited, Auditor=Read
  - Events: all read; create/edit restricted
  - Governance: Finance=Limited (expenses only)

### 3.18 Navbar Context Processor (Doc 4 §2)
- **File**: NEW `apps/core/context_processors.py`
- **Function**: `persona_nav_items(request)` — returns nav items based on user's active personas per Doc 4 §2 matrix
- **Register** in `config/settings.py` → `TEMPLATES[0]['OPTIONS']['context_processors']`

### 3.19 Admin Registrations
- Register all new models in their respective `admin.py` files:
  - `EventMilestone` in `apps/events/admin.py`
  - `Program`, `ProgramMilestone` in `apps/programs/admin.py`
  - `UserRoleRequest` in `apps/stakeholders/admin.py`
  - `UploadAuditLog` in `apps/core/admin.py`

### 3.20 Management Commands
- **Update** `apps/core/management/commands/load_demo_data.py`:
  - Add demo programs with milestones
  - Add demo event milestones on existing fundraising events
  - Add demo role requests (pending + approved)
  - Add demo upload audit log entries

---

## Phase 4 — Settings & Requirements

### 4.1 requirements.txt Additions
```
django-simple-history>=3.5
python-magic>=0.4
pyclamd>=0.4
Pillow>=10.0
```

### 4.2 config/settings.py Additions
```python
# New app
"apps.programs.apps.ProgramsConfig",
# Audit trail
"simple_history",
# SES email (production)
AWS_SES_REGION_NAME = env("AWS_SES_REGION", default="us-east-1")
# File security
UPLOAD_MAX_DOCUMENT_BYTES = 10 * 1024 * 1024  # 10MB
UPLOAD_MAX_PHOTO_BYTES = 5 * 1024 * 1024  # 5MB
CSV_UPLOAD_MAX_ROWS = 500
CLAMAV_HOST = env("CLAMAV_HOST", default="localhost")
CLAMAV_PORT = env.int("CLAMAV_PORT", default=3310)
```

---

## Phase 5 — Future Enhancements (Doc 1 §7, Doc 2 §7)

### 5.1 Stripe / PayPal Integration
- New `Payment` model: amount, currency, stripe_payment_id, status, donor, pool
- Webhook endpoint for Stripe callbacks
- Auto-create `Contribution` record on successful payment
- Add `stripe` to requirements

### 5.2 Donor Portal
- New views: giving history, project impact metrics, tax receipt download
- PDF generation for receipts (80G, Form 10BD for CSR donors)

### 5.3 Student / Faculty View
- Read-only project views filtered by facility/department
- No financial data exposed
- Feedback/photo submission as milestone evidence

### 5.4 Analytics Dashboard
- Aggregate views: donation trends, project completion rates, pool utilization
- Chart.js / Plotly integration via JSON API endpoints

### 5.5 Django REST Framework API (Phase 4)
- DRF serializers for all models
- API endpoints for mobile app consumption

### 5.6 AWS S3 + django-storages (Phase 2)
- Update `MEDIA_ROOT` to S3 bucket
- Add quarantine bucket for flagged uploads
- Configure `django-storages` with boto3

---

## Migration & Execution Order

| Step | Action | Dependencies |
|------|--------|-------------|
| 1 | Add new model fields to existing models (1.1–1.5, 1.10, 1.12) | None |
| 2 | Create `apps.programs` app (1.8, 1.9, 2.x) | Step 1 |
| 3 | Create EventMilestone model (1.6, 1.7) | Step 1 |
| 4 | Create UploadAuditLog model (1.11) | None |
| 5 | Run all migrations | Steps 1–4 |
| 6 | Implement upload security pipeline (3.11) | Step 4 |
| 7 | Implement CSV verification (3.3) | Step 6 |
| 8 | Implement funding filters/pagination (3.1) | None |
| 9 | Implement pool balance logic (3.5) | None |
| 10 | Implement rollup alerts (3.4) | None |
| 11 | Implement profile expansion (3.7) | Step 1 |
| 12 | Implement role request workflow (3.13) | Step 1 |
| 13 | Implement event milestone views (3.14, 3.15) | Step 3 |
| 14 | Implement program views (2.2–2.4) | Step 2 |
| 15 | Implement rollup programs section (3.16) | Step 14 |
| 16 | Implement permissions matrix (3.17) | All above |
| 17 | Implement navbar context processor (3.18) | Step 16 |
| 18 | Implement email notifications (3.8) | Step 12 |
| 19 | Implement audit trail (3.9) | None |
| 20 | Configure Google OAuth (3.10) | None |
| 21 | Update admin registrations (3.19) | All models |
| 22 | Update demo data command (3.20) | All above |
