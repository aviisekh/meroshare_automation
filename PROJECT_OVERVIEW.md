# Project Overview

This repository is a **Cypress automation project** that logs into [meroshare.cdsc.com.np](https://meroshare.cdsc.com.np/) and applies for available IPOs.

## What it does

1. Reads credentials and runtime values from environment variables (`.env`).
2. Opens MeroShare and logs in.
3. Navigates to **My ASBA**.
4. Fetches applicable share issues via API intercept.
5. Filters issues to only:
   - IPO
   - Ordinary Shares
   - For General Public
   - Not already applied
6. For each filtered issue, opens the apply dialog and submits an IPO application.

## Main files

- `cypress/e2e/meroshare/meroshare.cy.js` 
  - End-to-end test flow and application logic.
- `cypress/support/commands.js`
  - Custom Cypress command for login.
- `cypress.config.js`
  - Cypress base URL and environment variable loading.
- `.github/workflows/schedule-run.yaml`
  - Scheduled and manual CI workflow that runs Cypress across multiple user profiles and posts Telegram notifications.
- `.env.example`
  - Template for required runtime variables.

## How to run locally

```bash
npm install
cp .env.example .env
# edit .env with real values
npm run cypress:open   # interactive mode
npm run cypress:run    # headless mode
```

## Notes and caveats

- The script currently asserts success by expecting the apply endpoint to return HTTP `201`.
- `MAX_IPO_PRICE` is captured in test config but is not used in the currently active apply path.
- GitHub Actions matrix users are hard-coded in `.github/workflows/schedule-run.yaml`; each matrix key must exist as a repository secret containing a full `.env` payload.
- Current artifact delivery is Telegram-first (CI uploads Cypress videos to Telegram), so there is no in-app \"latest run artifact\" viewer yet.
