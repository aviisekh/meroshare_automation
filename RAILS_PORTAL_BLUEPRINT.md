# Rails Portal Blueprint for MeroShare IPO Automation

This blueprint adds a Rails-based multi-account portal on top of the existing Cypress automation.

## Goals

- Manage many MeroShare profiles from a web UI.
- Encrypt and store credentials safely.
- Schedule each profile to run daily (or custom cron).
- Execute Cypress per profile and keep execution logs/results.

## Recommended Stack

- **Rails 8** (or Rails 7.1+) with PostgreSQL
- **Devise** for portal authentication
- **Sidekiq + Redis** for background jobs
- **sidekiq-cron** for recurring schedules
- **Active Record Encryption** for sensitive profile fields
- Existing Node/Cypress project invoked from Rails jobs

## Domain Model

### users
Portal login users.

- id
- email
- encrypted_password
- role (admin/member)
- timestamps

### meroshare_profiles
A user's automation profile.

- id
- user_id (FK)
- label
- username (encrypted)
- password (encrypted)
- dp (encrypted)
- transaction_pin (encrypted)
- crn (encrypted)
- bank_name
- max_ipo_price (decimal)
- kitta (integer, default 10)
- active (boolean, default true)
- timezone (default `Asia/Kathmandu`)
- timestamps

### schedules
Per-profile schedule settings.

- id
- meroshare_profile_id (FK)
- cron_expression (e.g. `45 6 * * *` UTC)
- next_run_at
- active (boolean)
- timestamps

### automation_runs
Execution history.

- id
- meroshare_profile_id (FK)
- status (`queued`, `running`, `success`, `failed`, `skipped`)
- started_at
- finished_at
- exit_code
- summary
- log_path
- video_path
- screenshot_path
- ipo_found (boolean)
- applied (boolean)
- timestamps

## High-Level Flow

1. User creates profile in portal.
2. User sets daily schedule.
3. Scheduler enqueues `RunProfileAutomationJob`.
4. Job exports env vars and runs Cypress for that profile.
5. Output is captured into `automation_runs`.
6. Portal displays run history and status.

## Job Execution Pattern

Use a service object to isolate execution.

```ruby
# app/services/cypress_runner.rb
class CypressRunner
  def initialize(profile:, run:)
    @profile = profile
    @run = run
  end

  def call
    env = {
      "USER_NAME" => @profile.username,
      "PASSWORD" => @profile.password,
      "DP" => @profile.dp,
      "TRANSACTION_PIN" => @profile.transaction_pin,
      "CRN" => @profile.crn,
      "MAX_IPO_PRICE" => @profile.max_ipo_price.to_s,
      "BANK_NAME" => @profile.bank_name,
      "KITTA" => @profile.kitta.to_s
    }

    cmd = "npm run cypress:run"
    output = +""
    status = nil

    Open3.popen2e(env, cmd, chdir: Rails.root.join("..")) do |_stdin, stdout_err, wait_thr|
      stdout_err.each { |line| output << line }
      status = wait_thr.value
    end

    [status.exitstatus, output]
  end
end
```

## Suggested Routes

```ruby
Rails.application.routes.draw do
  devise_for :users
  root "dashboard#index"

  resources :meroshare_profiles do
    resource :schedule, only: %i[show create update destroy]
    resources :automation_runs, only: %i[index show create]
    post :run_now, on: :member
  end

  require "sidekiq/web"
  authenticate :user, ->(u) { u.role == "admin" } do
    mount Sidekiq::Web => "/sidekiq"
  end
end
```

## Security Notes

- Never log raw credentials.
- Encrypt sensitive DB fields.
- Restrict profile access to owner/admin.
- Use Rails credentials or secret manager for encryption keys and telegram secrets.

## Deployment Notes

- Run web + sidekiq + redis.
- Keep Cypress/Chrome dependencies available in worker image.
- Persist artifacts (`cypress/videos`, screenshots, logs) via mounted volume or object storage.

## Incremental Delivery Plan

1. Bootstrap Rails app + Devise + profile CRUD.
2. Add encrypted profile fields + validation.
3. Add Sidekiq + manual `Run now` action.
4. Add recurring scheduling via sidekiq-cron.
5. Add run history UI and artifacts.
6. Add telemetry/alerts (Telegram + email).
