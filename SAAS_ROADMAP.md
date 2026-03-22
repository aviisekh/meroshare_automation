# SaaS Product Blueprint: Multi-Profile MeroShare IPO Automation (Rails)

This document translates your idea into a practical Rails SaaS architecture.

## Product Goal

Build a SaaS where:
- A **user** can sign in and manage their account.
- Each user can create and manage **multiple MeroShare profiles** (different DP/login/CRN/bank settings).
- Each profile can have its own **schedule (cron-like)** to check IPO availability and auto-apply based on profile rules.
- The platform tracks history, job outcomes, and notifications.

---

## Recommended Tech Stack (Rails)

- **Framework**: Rails 8 (or Rails 7.1+) + PostgreSQL.
- **Auth**: Devise (email/password) + optional 2FA.
- **Background jobs**: Sidekiq + Redis.
- **Recurring jobs**: sidekiq-cron (or GoodJob recurring if you prefer Postgres-only jobs).
- **Secrets encryption**: Rails encrypted attributes (`ActiveRecord::Encryption`) for stored credentials.
- **Browser automation service**:
  - Option A: keep Cypress runner as an external worker container.
  - Option B (long-term): Playwright service per job.
- **Notifications**: Telegram first, then email/webhooks.
- **Billing**: Stripe (for profile limits / higher run frequency / alerts).

---

## Core Domain Model

## 1) `users`
Platform accounts.

Suggested fields:
- `email` (unique)
- `encrypted_password` (Devise)
- `time_zone` (default user timezone)
- `plan` (free/pro)
- `status` (active/suspended)

## 2) `profiles`
Represents one MeroShare account/config under a user.

Suggested fields:
- `user_id`
- `name` (e.g., "Dad MeroShare", "Personal")
- `dp_name`
- `mero_username` (encrypted)
- `mero_password` (encrypted)
- `transaction_pin` (encrypted)
- `crn` (encrypted)
- `bank_name`
- `default_kitta`
- `max_ipo_price`
- `enabled` (bool)

## 3) `schedules`
Cron settings per profile.

Suggested fields:
- `profile_id`
- `cron_expression` (e.g., `*/20 5-11 * * 1-5`)
- `time_zone`
- `active` (bool)
- `next_run_at`

## 4) `job_runs`
Execution audit trail (every scheduled/manual check run).

Suggested fields:
- `profile_id`
- `trigger_type` (scheduled/manual/retry)
- `started_at`, `finished_at`
- `status` (queued/running/success/failed/partial)
- `summary` (jsonb)
- `error_message`
- `log_excerpt`

## 5) `ipo_events`
What was discovered and what happened for each issue.

Suggested fields:
- `job_run_id`
- `company_name`
- `scrip`
- `share_type`
- `share_group`
- `sub_group`
- `issue_open_date`, `issue_close_date`
- `share_price`
- `decision` (skipped/applied/already_applied)
- `reason`

## 6) `notification_channels` (optional)
Per-user delivery settings.

Suggested fields:
- `user_id`
- `kind` (telegram/email/webhook)
- `config` (jsonb)
- `enabled`

---

## Multi-Tenant Boundaries (Critical)

- Every tenant-owned table must be scoped by `user_id` through associations.
- Enforce policy layer with Pundit/ActionPolicy.
- Never allow cross-user profile access by raw IDs.
- Add DB indexes for tenancy safety and speed:
  - `profiles(user_id, id)`
  - `schedules(profile_id, active)`
  - `job_runs(profile_id, started_at desc)`

---

## Scheduler + Worker Design

## Scheduler loop
1. A recurring system job wakes every minute.
2. It finds due `schedules` with `active = true` and `next_run_at <= now`.
3. It enqueues `RunProfileCheckJob(profile_id, schedule_id)` and updates `next_run_at`.

## Worker flow (`RunProfileCheckJob`)
1. Create `job_run` row (`running`).
2. Fetch profile config and decrypt secrets.
3. Invoke automation runner (Cypress/Playwright service).
4. Parse runner output into `ipo_events`.
5. Mark `job_run` as success/failed.
6. Send notifications (summary + failures).

---

## Integrating Your Existing Cypress Logic

Because your current repo already automates the MeroShare flow, use it as a **worker service** first:

- Rails app writes a payload per run (profile credentials + constraints).
- Worker container consumes payload and executes Cypress headless.
- Worker returns normalized JSON result:
  - discovered IPOs
  - skipped/applied decisions
  - response status codes
  - raw logs/video artifact links

This lets you launch fast without rewriting automation immediately.

---

## Suggested API / UI Surface

## Dashboard
- "Runs today", success rate, next scheduled run.

## Profiles
- Create/edit profile.
- Test connection / dry run.
- Enable/disable profile.

## Schedules
- Cron editor + timezone.
- Predefined templates (e.g., every 30m during market window).

## Runs & History
- List runs with status, duration, issues processed.
- Expand for event-level decisions and error logs.

## Notifications
- Telegram token/chat id setup.
- Toggle success/failure alerts.

---

## Security Requirements (Non-Negotiable)

- Encrypt all sensitive profile credentials at rest.
- Redact secrets from logs and UI.
- Add CSRF + brute-force protection on auth endpoints.
- Restrict background worker egress where possible.
- Record audit fields (`created_by`, `updated_by`, IP/user-agent for critical changes).
- Add user-consent/legal text because app performs financial actions.

---

## MVP Roadmap (Practical Sequence)

## Phase 1 (2–3 weeks)
- Auth + tenant model.
- Profile CRUD with encrypted fields.
- Manual "Run now" job per profile.
- Basic run history and logs.

## Phase 2 (2 weeks)
- Per-profile recurring schedules.
- Notification channels (Telegram).
- Retry strategy and failure alerting.

## Phase 3 (2 weeks)
- Billing plans + limits (profiles/schedules/run frequency).
- Team features (shared workspace).
- Better analytics and SLA dashboards.

---

## Starter Rails Commands

```bash
rails new ipo_saas -d postgresql
bundle add devise sidekiq redis sidekiq-cron pundit
rails generate devise:install
rails generate devise User
rails generate model Profile user:references name dp_name bank_name default_kitta:integer max_ipo_price:decimal enabled:boolean
rails generate model Schedule profile:references cron_expression time_zone active:boolean next_run_at:datetime
rails generate model JobRun profile:references trigger_type started_at:datetime finished_at:datetime status error_message:text summary:jsonb
rails generate model IpoEvent job_run:references company_name scrip share_type share_group sub_group issue_open_date:datetime issue_close_date:datetime share_price:decimal decision reason:text
```

---

## First Milestone Definition of Done

- User can sign up and add 1+ profiles.
- User can click "Run now" and see a completed run record.
- Each run stores per-IPO decisions and status.
- Failure path produces actionable error logs.
- No plaintext secrets in database/log output.
