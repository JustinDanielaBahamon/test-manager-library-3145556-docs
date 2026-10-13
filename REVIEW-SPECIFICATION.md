# Specification Review — Manager Library

## Overview
This repository contains the specification for the Manager Library system, to be implemented with **React** (frontend) and **PHP/Laravel** (backend). The original specification is written for Python/.NET, but the business rules, data model, and API contract must be preserved and translated to the new stack.

## Reviewed Documents (in `spec-python/` and `spec-.net/`)

### 1. `constitution.md`
**Non-negotiable principles:**
- Domain independence (no framework dependencies)
- Ports/declarations pattern (application knows domain, not adapters)
- Single composition module (bootstrap binds ports to adapters)
- SOLID principles (SRP, OCP, LSP, ISP, DIP)
- Schema ownership by migrations (infra provides empty DB, DDL in app migrations)
- Base enforces invariants (CHECK, UNIQUE, FK constraints)
- Derived states calculated, not stored (loan status, days late)
- Frozen values at loan time (title, ISBN, category freeze)
- No secrets in repo (env vars only, `.env.example` versioned)
- Document prevails over data (if discrepancy, fix the document)
- English for code/columns; Spanish for user docs/messages
- Evidence required for task completion

### 2. `spec.md`
**System purpose:** Library management for small library: catalog, members, loans, returns, overdue reports.

**Actors:**
- `admin`: maintains catalog and members, appoints librarians, views all reports
- `librarian`: registers loans/returns, queries catalog/subscribers/overdue
- `member`: borrows books (no login; identified by document number)

**Fixed scope:**
- 5 fixed categories (Novela, Ciencia, Tecnología, Historia, Infantil)
- No: fines, payments, reservations, renewals, email/SMS, multi-branch loans

**User Stories & Acceptance Criteria (summary):**
- HU-01: Login with token (8h validity), 401 without credentials, 401/403 on protected endpoints
- HU-02: Admin creates librarians; username lowercased/trimmed, unique; password never returned/logged
- HU-03: CRUD books; ISBN 13 unique among active; total_copies >= 1; logical delete (keeps history)
- HU-04: Search books by text (title/author), category filter, active only, paginated (max 50)
- HU-05: Create members; document number unique; no member deletion
- HU-06: Register loan (1-3 items per loan, no duplicate books), max 3 books/member, overdue members can't borrow, due date = loaned_at + 14 days, item data freezes at loan time, immutable loan
- HU-07: Register return (one line at a time), can't return twice, available copies never exceed total, loan closed when all lines returned
- HU-08: Query loan with calculated status (open/overdue/closed), list loans by date/status/member, paginated
- HU-09: List overdue lines (due_at < now), ordered by days late, admin/librarian only
- HU-10: Report most borrowed by date range; one row per book+category (frozen); no sibling/bibliotecario breakdown; frozen data preserved even if book renamed

### 3. `data-model.md`
**6 tables (schema `library`):**
- `category`: id (uuid, PK), name (varchar 120, unique) — seed with 5 categories
- `book`: id (uuid, PK), title (varchar 250), author (varchar 200), isbn (char 13, unique partial where deleted_at IS NULL), category_id (uuid, FK→category, RESTRICT), total_copies (integer ≥ 1), available_copies (integer 0–total_copies), cover_key (varchar 512, nullable, opaque key), deleted_at (timestamptz, nullable = active)
- `member`: id (uuid, PK), full_name (varchar 200), document_number (varchar 30, unique)
- `loan`: id (uuid, PK), member_id (uuid, FK→member, RESTRICT), loaned_by_user_id (uuid, FK→user, RESTRICT), loaned_at (timestamptz), returned_at (timestamptz, nullable)
- `loan_item`: id (uuid, PK), loan_id (uuid, FK→loan, CASCADE), book_id (uuid, FK→book, RESTRICT), book_title (varchar 250, frozen), isbn (char 13, frozen), category_name (varchar 120, frozen, no FK!), due_at (timestamptz = loaned_at + 14d), returned_at (timestamptz, nullable)
- `user`: id (uuid, PK), username (varchar 120, unique, lowercased/trimmed), password_hash (varchar 512, never indexed, never in response), role (varchar 40, only admin/liberian)

**Key constraints (by mark):**
- **Motor (base):** PK, FK, unique indexes, CHECK constraints, partial unique index on isbn
- **Solo dominio (enforced in code):** max 3 books/member, overdue member can't borrow, loan/item invariants
- **Pendiente (T-xx):** to be lowered to base when task implemented (ck_book_total_positive, ck_user_role)

**Time policy:** All timestamps UTC timestamptz. No created_at/updated_at.

**Seed:** 5 categories with fixed UUIDs in initial migration. Admin user created from env variables. Demo tool seeds via API.

### 4. `api-contract.md`
**Base:** `/api/v1`
**Format:** JSON, UTF-8, snake_case field names
**Auth:** `Authorization: Bearer <token>` on all endpoints except POST /auth/login and GET /health
**Pagination:** `?page=1&size=20`, size max 50; response `{ items, page, size, total }`

**Error format:**
```json
{ "code": "BOOK_NOT_AVAILABLE", "message": "No quedan ejemplares...", "details": [...] }
```

**Error codes:**
- 401: INVALID_CREDENTIALS, token issues
- 403: FORBIDDEN_ROLE
- 404: NOT_FOUND
- 409: USERNAME_TAKEN, ISBN_TAKEN, DOCUMENT_TAKEN, BOOK_HAS_OPEN_LOANS, BOOK_NOT_AVAILABLE, MEMBER_HAS_OVERDUE, ITEM_ALREADY_RETURNED
- 422: MEMBER_LIMIT_EXCEEDED, DUPLICATE_BOOK_IN_LOAN, VALIDATION_ERROR, INVALID_DATE_RANGE

**Endpoints:**

| Category | Method | Route | Role |
|---|---|---|---|
| Auth | POST /auth/login | público | Login |
| Auth | GET /health | público | Health check |
| Users | POST /users | admin | Create librarian |
| Categories | GET /categories | autenticado | List categories |
| Books | GET /books | autenticado | Search (q, category_id, paginated) |
| Books | GET /books/{id} | autenticado | Detail |
| Books | POST /books | admin | Create |
| Books | PUT /books/{id} | admin | Replace data |
| Books | DELETE /books/{id} | admin | Logical delete |
| Books | PUT /books/{id}/cover | admin | Upload cover (JPG/PNG, max 2MB) |
| Members | GET /members | admin,librarian | List/search (q, paginated) |
| Members | GET /members/{id} | admin,librarian | Detail with active_loans count |
| Members | POST /members | admin,librarian | Create member |
| Loans | POST /loans | admin,librarian | Register loan (1-3 items) |
| Loans | GET /loans | admin,librarian | List (from/to/status/member, paginated) |
| Loans | GET /loans/{id} | admin,librarian | Detail with lines+status |
| Loans | POST /loans/{loan_id}/items/{item_id}/return | admin,librarian | Return line |
| Loans | GET /loans/overdue | admin,librarian | Overdue lines |
| Reports | GET /reports/most-borrowed | admin | Date range required; rows: book_id, book_title, category_name, times_borrowed |

**Security:**
- JWT with sub and role, 8h validity, no personal data inside
- Passwords only hash, never in response/logs
- Authorization per-endpoint; 403 doesn't reveal resource existence
- CORS: only configured frontend origin
- Cover upload: only image/jpeg/image/png, max 2MB, validated by content

### 5. `tasks.md`
**Key tasks (SDD order):**
- T-01: Project skeleton + GET /health
- T-02: Initial migration (6 tables, FK, unique indexes, 5 categories seeded)
- T-03: Domain value objects (Isbn, Quantity, DateRange)
- T-04: Domain Book (Withdraw, Restock, logical delete)
- T-05: Domain Loan (AddItem, ReturnItem, calculated status, 14d, max 3)
- T-06: Domain Member/User (name normalization, role validation)
- T-07: Auth ports & use cases (LoginService, CreateLibrarianService) with fakes
- T-08: Books ports & use cases (create, search, edit, delete, cover) with fakes
- T-09: LendBooksService with fakes (limits, overdue, no stock)
- T-10: Return service + overdue service
- T-11: Most-borrowed report + its read port
- T-12: Persistence adapters (book, member, user repos) vs PostgreSQL
- T-13: Loan repository + version marker (RNF-2: concurrent last copy)
- T-14: Security adapter (password hash, JWT token) — never in logs/responses
- T-15: Cover adapter (disk) — 2MB ok, 3MB → 422, none → cover_key null
- T-16: HTTP adapter (all routes, error bodies, role auth) — contract test
- T-17: Composition + admin initial from env — system starts with empty DB, can login as admin
- T-18: Lower base invariants (ck_book_total_positive, ck_user_role)
- T-19: Architecture tests (import-linter, prohibited imports)
- T-20: Infra (docker compose up) — empty DB, API, frontend
- T-21: Tool — demo data via API (≥40 books, 10 members, 15 loans, 3 overdue)
- T-22: App — full UI: login, catalog, members, loan, return, overdue, report
- T-23: Page — static presentation site

**Done when:** T-17 with evidence has the core. T-18→T-23 separate good from excellent. SDD performance evaluated, not task count.

### 6. `constitution.md` Article XIII (implied)
**Language:** Code, columns, and paths in English. User-facing messages in Spanish.

## Translation Notes for Laravel/React Stack

| Spec Concept | Laravel Equivalent | React Equivalent |
|---|---|---|
| PostgreSQL timestamptz | `Carbon::now('UTC')` or `now()` | ISO strings, format with `Z` |
| JWT auth (`sub`, `role`) | `laravel/sanctum` or `laravel/jwt` | Token stored in localStorage/cookies |
| Snake_case JSON | Resource transformers | camelCase components, API snake_case requests |
| Partial unique index on isbn | Migration: `->uniqueIndex('isbn', 'WHERE deleted_at IS NULL')` | Validation on create/update |
| Frozen values at loan time | Save copy in loan_item before INSERT | Store in state, never read from live book |
| Max 3 books/member | Custom validation rule or Eloquent observer | Check in component before loan request |
| OPAQUE cover_key | Store just the key (S3 path, disk path hash) | Request `GET /books/{id}/cover` for binary |

## Next Steps
1. Translate business rules to Laravel Eloquent models and controllers
2. Translate API endpoints to Laravel routes with Sanctum/JWT auth
3. Implement React components following the API contract
4. Preserve all frozen values and invariants as documented
5. Run contract tests against the implemented API