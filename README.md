# Runrbee Backend

The API and business rules for Runrbee: a location-based app where customers choose a verified runner near them to run errands, with payment held until the errand is confirmed.

> **Status: early development.** This README describes the intended setup. The commands below will work once the matching files exist (the Docker Compose file, `.env.example`, migrations, and the health route are part of the first milestone). Update this file as you build.

---

## What this service does

- Phone number and one-time-code login with rotating tokens
- Errands: post, get applications from runners, request a specific runner, choose one
- Location: availability by runner density, nearby errands, live runner tracking
- In-app chat, proof photos, and handover codes
- Payments held until confirmation, refunds, and runner payouts (gateway to be chosen)
- KYC levels for runners and customers (provider to be chosen)
- Two-way ratings with blind reveal
- Safety: SOS, trusted contacts, check-ins, trip sharing
- Admin: runner approval, disputes, flags, review cases

## Tech stack

| Part | Choice |
|---|---|
| Language | Rust (stable) |
| Web framework | Axum on Tokio |
| Database access | SQLx with SQL migrations |
| Database | PostgreSQL with PostGIS |
| Local environment | Docker and Docker Compose |

Other crates you will likely use: `serde`, `tower-http`, `tracing`, `argon2`, `jsonwebtoken`, `uuid`, `validator`, `reqwest`, and `utoipa` (for OpenAPI).

## Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) (install with `rustup`)
- [Docker](https://docs.docker.com/get-docker/) with Docker Compose
- `sqlx-cli` for migrations:

```bash
cargo install sqlx-cli --no-default-features --features rustls,postgres
```

Optional but recommended:

```bash
cargo install cargo-audit    # checks dependencies for known vulnerabilities
```

## Getting started

```bash
# 1. Clone
git clone https://github.com/Runrbee/r-backend.git
cd r-backend

# 2. Create your local config (never commit .env)
cp .env.example .env
# then edit .env and replace the placeholder secrets

# 3. Start the database (PostgreSQL with PostGIS)
docker compose up -d db

# 4. Run migrations
sqlx migrate run

# 5. Run the API
cargo run

# 6. Check it is up (once the health route exists)
curl http://localhost:8080/health
```

## Configuration

Settings come from environment variables. Copy `.env.example` to `.env` for local work.

```bash
# .env.example  (placeholders only)
APP_ENV=development
APP_HOST=0.0.0.0
APP_PORT=8080
RUST_LOG=info

DATABASE_URL=postgres://app:change_me@localhost:5432/errands

# Generate each of these with: openssl rand -base64 48
JWT_SECRET=replace-me
OTP_HASH_KEY=replace-me

# Generate with: openssl rand -base64 32
# Encrypts NIN, BVN, and bank account numbers stored in the database
DATA_ENCRYPTION_KEY=replace-me

# Fill in when you choose providers
SMS_PROVIDER=
SMS_API_KEY=
PAYMENT_PROVIDER=
PAYMENT_SECRET_KEY=
PAYMENT_WEBHOOK_SECRET=
KYC_PROVIDER=
KYC_API_KEY=
KYC_WEBHOOK_SECRET=

# Photo storage (S3-compatible)
STORAGE_ENDPOINT=
STORAGE_BUCKET=
STORAGE_ACCESS_KEY=
STORAGE_SECRET_KEY=
```

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Connection string for PostgreSQL |
| `JWT_SECRET` | Signs access tokens |
| `OTP_HASH_KEY` | Used when hashing one-time codes |
| `DATA_ENCRYPTION_KEY` | Encrypts sensitive identity and bank data at rest |
| `SMS_*`, `PAYMENT_*`, `KYC_*` | Provider settings, filled in once each provider is chosen |
| `STORAGE_*` | Private storage for proof photos and selfies |

**Never commit `.env`.** Make sure `.env` is in `.gitignore`.

## Project layout

```
.
├── Cargo.toml
├── docker-compose.yml
├── .env.example
├── migrations/          SQL migrations (one per schema section)
├── docs/                PRD, API, schema, and decision records
└── src/
    ├── main.rs
    ├── config.rs
    ├── error.rs         shared error type and JSON error format
    ├── routes/          auth, errands, runners, requests, chat, admin, webhooks
    ├── domain/          errand status rules, refund and fee rules, ledger rules
    ├── services/        otp, tokens, payments, kyc, storage, notifications
    └── db/              queries
```

## Database and migrations

The schema lives in `migrations/` and follows the order in `docs/schema.md`: setup, accounts, runners, location and errands, chat and disputes, payments, audit, safety, KYC and ratings, direct requests.

```bash
sqlx migrate add <name>     # create a new migration
sqlx migrate run            # apply pending migrations
sqlx migrate revert         # undo the last migration (if it has a down file)
```

Rules:

- **Never edit a migration that has already run.** Add a new one.
- The local database uses the `postgis/postgis` image so the `postgis` extension is available.
- Money is stored as integer kobo, locations as `geography(Point, 4326)`, and times as `timestamptz`.

## Running tests and checks

```bash
cargo test                          # unit and integration tests
cargo fmt --all                     # format
cargo clippy --all-targets -- -D warnings   # lint
cargo audit                         # dependency vulnerabilities
```

Integration tests need a test database. Use a separate database (for example `errands_test`) and never point tests at real data.

## Rules the code must keep

These come from the design documents. Break them and the app becomes unsafe or inconsistent.

- **Errand status changes happen in one place.** One function decides which transitions are allowed, and it has tests.
- **One runner per errand.** Choosing a runner (from an application or a request) uses a guarded database update, so only one path can win.
- **Money is append-only.** Payments go through an internal ledger that is never edited. Corrections are new entries. Amounts are integer kobo.
- **Ratings never decide payouts.** Money releases on confirmation, the end of the confirmation window, or an admin decision.
- **Providers sit behind interfaces.** Payments and KYC use traits with one adapter per provider, so providers can be swapped.
- **Never trust a provider without checking.** Verify webhook signatures and process each event once.
- **Protect identity data.** NIN, BVN, and account numbers are encrypted at rest. Do not log them, tokens, or phone numbers.
- **Hide exact addresses** from runners until they are chosen and payment is held.
- **Policy numbers are data.** Refund percentages and timers live in the `policy_settings` table, not scattered in code.

## API documentation

See `docs/api.md` for the endpoint list, status transitions, and rules. Once the code exists, generate an OpenAPI spec from it (for example, with `utoipa`) and treat that as the source of truth.

## Documentation

| File | Contents |
|---|---|
| `docs/prd.md` | Product requirements |
| `docs/api.md` | API and feature documentation |
| `docs/schema.md` | Database schema and key queries |
| `docs/adr.md` | Architecture decision records |

When you change a decision that affects the product, the API, or the schema, update the matching document and add or update a decision record.

## Roadmap

- [ ] **M0 Foundations:** project runs locally, database up, phone and OTP login
- [ ] **M1 Errands and matching:** post an errand, runners apply, customers request runners, customer picks one
- [ ] **M2 Tracking, chat, and safety:** live location, proofs, chat, SOS and check-ins
- [ ] **M3 Payments:** hold, release, and refund in the gateway sandbox
- [ ] **M4 KYC, admin, and ratings:** runner verification, admin tools, two-way ratings
- [ ] **M5 Closed test:** staging deployment, legal and security review, real test errands
- [ ] **M6 Review:** results compared with targets, go or no-go decision

## Contributing

- Work on a branch named like `feature/short-description` or `fix/short-description`.
- Keep pull requests small and focused.
- Before opening a pull request, run `cargo fmt`, `cargo clippy`, and `cargo test`, and add tests for new rules.
- Describe any migration, config change, or document update in the pull request.
- Do not commit secrets, real user data, or `.env` files.

## Security

Please do not report security problems in public issues. Email **onuhtheola3@gmail.com** with the details.

This service handles money and personal identity data. Have an experienced developer review the authentication, payments, and KYC code, and have a lawyer review the legal points, before real users use it.

## License

Not yet licensed for reuse. All rights reserved by Runrbee.
