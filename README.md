# Zambia Finance Leasing Platform — Backend (Phase 1 + 2)

A working Node.js/Express API implementing the core loan lifecycle: client
onboarding, KYC, collateral, loan applications, admin approval/disbursement,
the 30%-compounding interest engine, manual repayments, and automated
T-5/T-2/T-0 reminders. Runs on SQLite — no external database or accounts
needed to try it out. Uses Node's **built-in** SQLite module (`node:sqlite`,
available from Node 22+), not the `better-sqlite3` package — this avoids a
native compilation step entirely, so `npm install` works even on newer Node
versions (like Node 24) without needing build tools or extra network access.

This covers **Phase 1 and Phase 2** of the build plan (core workflow +
interest engine/reminders). **Phase 3 (live payment gateway)** and the
**frontend** are not included — see "What's not here yet" below.

## Requirements

- Node.js 22.5 or later (this includes Node 24). No database software, no
  build tools, no compiler needed — the whole thing runs off Node itself.

## Quick start

```bash
npm install
cp .env.example .env      # edit JWT_SECRET before going anywhere near production
npm run seed               # creates a super admin + a default loan product
npm start                  # starts the API on http://localhost:4000
```

The seed script prints the admin login (phone `0000000000`, password
`ChangeMe123!`) — **change this password immediately** in any real deployment;
there's no "change password" endpoint yet, so for now update it directly in
the database or re-run a modified seed.

## Verify it works

A full end-to-end test is included that exercises the entire loan lifecycle —
registration, KYC, collateral, application, approval, disbursement, a missed
due date, interest rollover, and full repayment — and prints the actual vs.
expected numbers at each step:

```bash
node test/e2e.js
```

It proves the interest math: a K10,000 loan becomes K13,000 at the first due
date (30%), and if missed, rolls over to K16,900 (30% compounding on the
unpaid balance, not the original principal) — exactly as specified.

## Project structure

```
src/
  db.js                  Schema + SQLite connection (auto-creates tables on first run)
  server.js               Express app entry point
  seed.js                  Creates the first admin user + a default loan product
  middleware/auth.js       JWT auth + role-based guards
  routes/
    auth.js                Register / login
    kyc.js                  Upload + admin review of NRC / proof of residence
    collateral.js            Declare + admin review of collateral
    loanProducts.js           Configure interest rate, period, min/max, collateral multiplier
    loans.js                   Apply, approve/reject, disburse, view, repay
    admin.js                    Loan book, collections queue, audit log, manual job trigger
  services/
    interestEngine.js         THE core logic: cycle creation, 30% compounding rollover, payment application
    reminderService.js         Computes T-5/T-2/T-0 and dispatches via opted-in channels
    audit.js                    Writes to the audit_log table
    notifiers/
      sms.js, whatsapp.js, email.js   Console stubs — see comments for how to plug in real providers
  jobs/
    dailyJob.js                Runs rollover + reminders once a day (cron) or on demand
test/
  e2e.js                       Full lifecycle smoke test (see above)
```

## The interest engine, in plain terms

This is the part that matters most to get right, so it's isolated in one file:
`src/services/interestEngine.js`.

- On disbursement: `closing_balance = principal × 1.30`
- On a missed due date: `new_closing_balance = old_unpaid_balance × 1.30`
  (compounds on whatever's still owed — principal + unpaid interest — not
  the original principal)
- On payment: reduces `amount_paid` on the currently open cycle; when it
  reaches the closing balance, the loan is marked `repaid` and the linked
  collateral is automatically marked `released`
- The daily job (`src/jobs/dailyJob.js`) finds every open cycle whose due
  date has passed and rolls it over — this is the automation that replaces
  someone manually chasing every loan every day

If you decide the 30% should apply to the **original principal** each time
instead of compounding (see the "open questions" in the project spec),
change one line in `rollOverCycle()` — it's isolated for exactly this reason.

## API overview

All endpoints are under `/api`. Authenticated routes need
`Authorization: Bearer <token>` from `/auth/login` or `/auth/register`.

| Area | Endpoint | Who |
|---|---|---|
| Auth | `POST /auth/register`, `POST /auth/login` | Public |
| KYC | `POST /kyc` (upload), `GET /kyc/mine`, `GET /kyc/pending`, `PATCH /kyc/:id/review` | Client / Admin |
| Collateral | `POST /collateral`, `GET /collateral/mine`, `GET /collateral/pending`, `PATCH /collateral/:id/review` | Client / Admin |
| Loan products | `GET /loan-products`, `POST /loan-products`, `PATCH /loan-products/:id` | All / Super admin |
| Applications | `POST /loans/applications`, `GET /loans/applications/mine`, `GET /loans/applications/pending`, `PATCH /loans/applications/:id/review`, `POST /loans/applications/:id/disburse` | Client / Admin |
| Loans | `GET /loans/mine`, `GET /loans/:id`, `POST /loans/:id/payments` | Client / Admin |
| Admin | `GET /admin/loans`, `GET /admin/collections`, `GET /admin/audit-log`, `POST /admin/run-daily-job` | Admin roles |

Roles: `client`, `loan_officer`, `collections`, `finance_admin`, `super_admin`.
Admin-only routes accept any of the four non-client roles unless noted.

## What's not here yet (and what to do about it)

- **Live payment gateway.** `POST /loans/:id/payments` is a manual/admin-
  recordable entry point right now — exactly what you want for Phase 1
  piloting. When you've chosen an aggregator (ZynlePay, DPO, etc.), replace
  this with a webhook handler that calls the same `applyPayment()` function
  in `interestEngine.js` once the provider confirms a transaction.
- **Real SMS/WhatsApp/Email.** The three notifier files are console-log
  stubs with the real provider code commented in as an example. Swapping
  them is a same-day task once you have API keys — nothing else in the app
  needs to change, since `reminderService.js` only calls `send()`.
- **Frontend.** This is a pure API. Build the client and admin web apps
  against the endpoints above, matching your Penpot designs.
- **Password reset, 2FA, rate limiting** — not implemented; add before
  production.
- **File storage.** KYC/collateral documents currently save to a local
  `uploads/` folder. Move this to an encrypted S3-compatible bucket before
  production, per the compliance notes in the project spec.
- **Postgres migration.** SQLite is used here for zero-config local
  development. For production, swap `node:sqlite` for `pg` and adjust the
  handful of SQLite-specific date functions in `admin.js` and
  `interestEngine.js` (`julianday`, `date('now', ...)`).
- **`node:sqlite` is still labeled experimental by Node.js** — it's been
  stable in practice and is used here specifically to avoid native-module
  build issues, but keep an eye on Node's release notes if you plan to run
  this in production long-term.

## Security notes before any real money moves through this

- Change `JWT_SECRET` in `.env` to a long random value.
- Change the seeded admin password immediately.
- Put this behind HTTPS in any real deployment.
- Encrypt `nrc_number` and uploaded documents at rest (see compliance notes
  in the main project spec).
- Get the legal review mentioned in the spec done before enabling real
  disbursements — this code implements the business rules as specified, but
  doesn't make them compliant on its own.
