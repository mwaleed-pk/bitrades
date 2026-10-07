<div align="center">

# 💹 Bitrades — Investment & Trading Platform

**Wallet-based investing, minus the passwords.** A full-stack investment platform with TRC20 wallet authentication, tiered investment plans, deposit/withdrawal workflows, a referral engine, and a complete admin operations panel.

![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=flat-square&logo=flask&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?style=flat-square&logo=sqlite&logoColor=white)
![HTML5](https://img.shields.io/badge/HTML5-E34F20?style=flat-square&logo=html5&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

</div>

---

## ✨ Features

- **Passwordless wallet auth** — users sign in with a TRC20 wallet address (validated against the `T…` Base58 format); first login auto-creates the account with a unique referral code
- **Tiered investment plans** — Basic, Premium, and Elite plans, each with its own minimum deposit and daily earning rate over a fixed 20-day term
- **Deposit workflow** — users submit deposits with amount + transaction ID; every deposit starts as `pending` until an admin approves or rejects it
- **Withdrawal workflow** — withdrawals gated by a minimum amount and a minimum-referral requirement to protect the reward pool; admin-approved processing
- **Referral engine** — every user gets a referral code; successful referred deposits earn the referrer a 2% commission, tracked in a dedicated referrals ledger
- **Live earnings math** — plan earnings accrue daily from the plan's rate and term; users see plan status via the `/api/plan_status` endpoint
- **Full admin panel** — dedicated dashboards for deposits, withdrawals, users (with balance adjustments), plans (manual completion), referrals, and an immutable admin action log
- **Audit trail** — every admin action (approvals, rejections, balance changes) is written to `admin_logs`

## 🛠️ Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | Python, Flask |
| Database | SQLite (`sqlite3`, row-factory access, per-request connections) |
| Templating | Jinja2 server-rendered pages |
| Frontend | Pure HTML/CSS/JS (no build step) |
| Auth | Flask sessions + `login_required` / `admin_required` decorators |
| Validation | Regex-based TRC20 address validation |
| Run | `run.py` bootstrap (initializes DB, then serves) |

## 🏗️ Architecture / How It Works

```
                    ┌────────────── USER ──────────────┐
                    ▼                                 ▼
              /auth (TRC20)                     /dashboard
                    │                                 │
        ┌───────────┴───────────┐           ┌─────────┴─────────┐
        ▼                       ▼           ▼                   ▼
  /deposit (submit         /withdraw   /api/plan_status   referral link
   txid + plan)            (gated)      (live earnings)    (?ref=CODE)
        │                       │
        └─────── pending ────────┘
                        ▼
              ┌──── ADMIN PANEL ────┐
              ▼        ▼        ▼   ▼
        /admin  /admin/  /admin/  /admin/
        deposits withdrawals users plans …
              │        │
        approve / reject → plans activated → daily earnings accrue → /admin/plan/complete
                        │
                   admin_logs (every action recorded)
```

**Data model:** `users` · `deposits` · `plans` · `withdrawals` · `referrals` · `admin_logs` — created idempotently by `init_db()` on startup.

**Money flow:** user submits deposit (amount + txid + plan) → `pending` → admin approves → plan activates with start/end timestamps → earnings accrue daily → admin can mark plans complete; withdrawals follow the same approve/reject pipeline.

## 🚀 Getting Started

### Prerequisites

- Python 3.10+

### Installation

```bash
git clone https://github.com/mwaleed-pk/bitrades.git
cd bitrades
python -m venv venv && source venv/bin/activate
pip install flask
```

### Environment variables

| Variable | Description |
|----------|-------------|
| `SECRET_KEY` | Flask session secret — **set this in production** (see Security Notes) |
| `DEPOSIT_ADDRESS` | Platform deposit wallet address shown to users |
| `ADMIN_WALLET` | Wallet address granted admin-panel access |

> Move all deployment-specific values out of source and into the environment before going live.

### Run

```bash
python run.py
# → initializes bitrade.db, then serves on http://0.0.0.0:5000
```

Open `http://localhost:5000`, enter a TRC20 wallet address to sign in. The admin panel lives at `/admin` and is restricted to the configured admin wallet.

## 📁 Project Structure

```
bitrades/
├── app.py                 # Flask app: auth, dashboard, deposits, withdrawals, admin, API
├── run.py                 # Bootstrap: chdir, init_db(), start server
├── auth.html              # Wallet sign-in page
├── base.html              # Shared layout
├── dashboard.html         # User dashboard (plans, earnings, referral link)
├── deposit.html           # Deposit submission form
├── admin.html             # Admin overview
├── admin_deposits.html    # Approve / reject deposits
├── admin_withdrawals.html # Approve / reject withdrawals
├── admin_users.html       # User list + balance adjustments
├── admin_plans.html       # Plan management + manual completion
├── admin_logs.html        # Admin action audit trail
└── admin_referrals.html   # Referral ledger
```

## 🔌 API Reference

The platform is primarily server-rendered; routes are page/form based:

| Method | Route | Description |
|--------|-------|-------------|
| `GET`  | `/` | Landing → redirects to dashboard or admin by session |
| `POST` | `/auth` | Wallet sign-in (validates TRC20 format, creates user on first login) |
| `GET`  | `/logout` | End session |
| `GET`  | `/dashboard` | User dashboard (login required) |
| `GET/POST` | `/deposit` | Submit a deposit (amount + txid + plan) |
| `POST` | `/withdraw` | Request a withdrawal (login required, gated) |
| `GET`  | `/api/plan_status` | JSON: live status/earnings of the user's plans |
| `GET`  | `/admin` | Admin overview (admin only) |
| `GET`  | `/admin/deposits` | Pending deposits queue |
| `POST` | `/admin/deposit/approve/<id>` · `/admin/deposit/reject/<id>` | Approve / reject a deposit |
| `GET`  | `/admin/withdrawals` | Pending withdrawals queue |
| `POST` | `/admin/withdrawal/approve/<id>` · `/admin/withdrawal/reject/<id>` | Approve / reject a withdrawal |
| `GET`  | `/admin/users` | User management |
| `POST` | `/admin/user/balance` | Adjust a user's balance (logged) |
| `GET`  | `/admin/plans` | Plan management |
| `POST` | `/admin/plan/complete/<id>` | Mark a plan complete |
| `GET`  | `/admin/logs` · `/admin/referrals` | Audit trail · referral ledger |

## 🔒 Security Notes

- **Session secret:** the Flask `secret_key` must come from an environment variable in production — never ship a hardcoded value.
- **Admin gating:** admin access is tied to a single configured wallet address and enforced by the `admin_required` decorator on every admin route.
- **Input validation:** wallet addresses are regex-validated as TRC20; amounts and transaction IDs should additionally be range/format-checked at the edge.
- **Config in env:** deposit addresses, commission rates, and thresholds are deployment configuration — keep them out of source control.
- **Database:** SQLite suits demos and single-node deploys; move to Postgres with proper backups before handling real funds.
- **HTTPS only:** always terminate TLS in front of the app; never expose the dev server directly.

## 🗺️ Roadmap

- [ ] Environment-driven config for all secrets and addresses
- [ ] On-chain deposit verification (instead of manual txid review)
- [ ] Postgres migration + automated backups
- [ ] Email/Telegram notifications for deposit & withdrawal events
- [ ] Rate limiting and CSRF protection on mutating routes
- [ ] Docker image + compose setup

## 🤝 Contributing

1. Fork the repo and create a feature branch
2. Keep the wallet-auth model and audit-log discipline intact
3. Open a pull request describing the change and its security implications

## 📄 License

MIT — see [LICENSE](LICENSE) for details.

## 👤 Author

**Muhammad Waleed (MW Trader)** — https://github.com/mwaleed-pk
