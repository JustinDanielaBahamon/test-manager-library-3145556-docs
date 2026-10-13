# Tasks — Laravel + React

Same tasks and dependencies as [`../spec-python/tasks.md`](../spec-python/tasks.md). Only the repository and the form of the
evidence change. A task is done when its evidence exists and can be run (Article XII). Initial state of every task: **pending**.

| # | Task | Repo | Evidence in this stack |
|---|---|---|---|
| T-01 | Project skeleton and `GET /api/v1/health` | api | `curl /api/v1/health` → `{"status":"ok"}`; container starts with one command |
| T-02 | Initial migration: six tables, FKs, unique indexes, five seeded categories | api | `php artisan migrate` on an empty DB; `data-model.md` §10 queries match; no `DEFAULT` |
| T-03 | Domain value objects `Isbn`, `Quantity`, `DateRange` | api | Unit tests: 13-digit ISBN, quantity > 0, end ≥ start |
| T-04 | Domain `Book` (withdraw, restock, soft delete) | api | Unit tests: never < 0, never > total |
| T-05 | Domain `Loan` (add item, return item, computed status, 14 days, max 3) | api | Unit tests with a fake `Clock`: `open`, `overdue`, `closed` |
| T-06 | Domain `Member` and `User` (name normalization, valid role) | api | `"Ana "` becomes `ana`; role `admin` through the API rejected |
| T-07 | Auth use cases (`LoginService`, `CreateLibrarianService`) with fakes | api | HU-01 and HU-02 pass against fakes |
| T-08 | Book use cases (create, search, edit, soft delete, cover) with fakes | api | HU-03 and HU-04 pass against fakes |
| T-09 | `LendBooksService` with fakes | api | HU-06: limits, overdue members, no copies |
| T-10 | Return and overdue use cases | api | HU-07 and HU-09; returning twice → `ITEM_ALREADY_RETURNED` |
| T-11 | Most-borrowed report and its read port | api | HU-10; old title kept after renaming (ADR-004) |
| T-12 | Persistence adapters: book, member, user repositories | api | Integration tests against real PostgreSQL |
| T-13 | Loan repository + last-copy protection | api | Two loans of the last copy: one 201, one 409 (RNF-2; see `spec-gaps.md` G-1) |
| T-14 | Security adapters: Argon2id hash and JWT | api | Password never in logs or responses (RNF-3) |
| T-15 | Cover adapter (local disk) | api | 2 MB PNG ok; 3 MB → 422; no cover → `cover_key` null |
| T-16 | HTTP adapter: contract routes, single error body, role authorization | api | Contract test over every route and every code of `api-contract.md` §2 |
| T-17 | Composition and boot (initial admin from environment) | api | Starts on an empty DB; admin can log in |
| T-18 | Move `ck_book_total_positive` and `ck_user_role` to the DB | api + docs | A violating `INSERT` via `psql` fails; `data-model.md` §4 updated |
| T-19 | Architecture tests (Articles I, II, III) | api | Test passes, and fails when a forbidden `use` is introduced |
| T-20 | `docker compose up` starts empty DB, API and frontend | infra | One command; DB has no tables until migrations run |
| T-21 | Demo seeder **through the API** (≥ 40 books, 10 members, 15 loans, 3 overdue) | tool | After running it, `/loans/overdue` returns 3 lines and the report has rows |
| T-22 | React screens: login, catalog, members, lend, return, overdue, report | app | Manual end-to-end walkthrough |
| T-23 | Static public site | page | Opens without the API; correct on mobile and desktop |

## Order of work for the delivery window

The spec says the core is T-01 to T-17, and that one task with evidence is worth more than three half done.
Work goes in this order, one branch and small commits per step:

1. `docs` — this adaptation (no code).
2. `api` — T-01 → T-02 → T-14/T-07/T-17 (login works) → T-08/T-12 → T-09/T-13 → T-10 → T-11 → T-16.
3. `infra` — T-20.
4. `app` — T-22, starting with login and catalog.
5. `page` — T-23.
6. `tool` — T-21.
