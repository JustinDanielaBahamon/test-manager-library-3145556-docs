# Methodology — SDD, DDD and TDD in this delivery

How the spec was turned into code, and what evidence exists for each part. Article XII applies: a task counts as done
only when its evidence exists and can be run. This file separates **verified**, **implemented (evidence pending)** and
**pending** on purpose.

## 1. SDD — Spec-Driven Development

The spec was read before writing code, in the order given by `README.md`: constitution → spec → plan/architecture →
data model and API contract → tasks → ADRs. Then:

| Step | Result |
|---|---|
| Choose the base spec | `spec-python/`. `spec.md`, `data-model.md` and `api-contract.md` were compared with `spec-.net/`: identical. |
| Translate only the technology | [`stack-mapping.md`](stack-mapping.md): each plan decision D-01 to D-12 mapped to Laravel/React. Business untouched. |
| Record what the spec does not say | [`spec-gaps.md`](spec-gaps.md): G-1 to G-12, each with the option that keeps tables and contract intact. |
| Plan by tasks | [`tasks-laravel.md`](tasks-laravel.md): T-01 to T-23 with repository and evidence. |
| Trace code to spec | Every domain class, service, repository and route group cites the story, criterion, rule or ADR it implements (e.g. `/** HU-06 */`, `ADR-002`, `CA-03.4`). |
| Small steps | One branch per task group, small commits, merge into `main` with `--no-ff`. |

The data model and the contract were treated as fixed: the migration reproduces `data-model.md` §3–§4 and §9 exactly,
and no table, column or endpoint was added. Where the spec and the data model disagreed (version token, G-1) the data
model won (Article X).

## 2. DDD — Domain-Driven Design

| Concept | Where |
|---|---|
| Ubiquitous language | The glossary of `data-model.md` §1 (book, copy, member, loan, loan line, frozen value) is used as is in class and column names. |
| Aggregates | `Book` (copies between 0 and total), `Loan` with its `LoanItem` lines (1–3 lines, no repeated book, 14-day due date, computed status). |
| Value objects | `Isbn` (13 digits), `DateRange` (end ≥ start, inclusive end). `Quantity` and `DocumentNumber` are validated inside `Book` and `Member` instead of separate classes. |
| Domain errors | `DomainError` carries the contract code (`MEMBER_LIMIT_EXCEEDED`, `DUPLICATE_BOOK_IN_LOAN`, …). The HTTP status is chosen by the HTTP adapter, not by the domain. |
| Frozen values | A loan line copies title, ISBN and category name at lending time (RN-4, ADR-004). |
| Computed state | `open` / `overdue` / `closed` is calculated by `Loan::statusOf`, never stored (Article VII). |

Architecture follows the hexagonal layout of `architecture.md`: `app/Domain` (plain PHP), `app/Application`
(ports + one service per use case), `app/Adapters` (HTTP, persistence, security, storage) and a single composition
point, `PortBindingsServiceProvider` (Article III).

## 3. TDD — Test-Driven Development

Honest status:

- **Domain unit tests exist and pass**: `api/tests/Unit/Domain` — 19 tests over `Isbn`, `Book`, `Loan`, `User` and
  `DateRange` (T-03 to T-06), using a fixed instant instead of waiting 14 days (D-05).
- The strict red → green → refactor cycle was **not** followed for every rule because of the time box; tests were written
  alongside the domain code, directly from the acceptance criteria.
- **Pending**: application tests with fakes (T-07 to T-11), integration tests against PostgreSQL (T-12, T-13),
  the contract test over every route and error code (T-16) and architecture tests (T-19).

## 4. Evidence status by task

| Status | Tasks | Evidence |
|---|---|---|
| **Verified** | T-02 schema | The migration SQL was run on PostgreSQL 16: 6 tables in `library`, 0 columns with `DEFAULT`, 5 categories. |
| **Verified** | T-03 to T-06 domain | 19 unit tests pass. |
| **Verified at SQL level** | T-13 last copy (RNF-2) | Two concurrent conditional updates on a book with 1 copy: one affects 1 row, the other 0 (→ 409). The `CHECK` rejects available > total. |
| **Verified at SQL level** | T-11 frozen report (ADR-004) | After renaming a lent book, the report query still returns the old title. |
| **Implemented, evidence pending run** | T-01, T-07 to T-10, T-12, T-14 to T-17, T-20 to T-23 | Code is in the repositories; the end-to-end run with `docker compose up` is the next evidence to collect. |
| **Pending** | T-18 (`CHECK`s), T-19 (architecture tests), RNF-1 load test | Not started. |
| **Blocked by a spec gap** | Part of T-21 (3 overdue lines) | See `spec-gaps.md` G-12. |
