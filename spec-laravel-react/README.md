# Manager Library — Laravel + React adaptation

This folder translates the **technology** decisions of the spec to the stack required by the challenge:
**PHP with Laravel** for the backend and **React** for the frontend.

It does **not** change the business. Stories, acceptance criteria, business rules, data model and API contract
stay exactly as written in [`../spec-python/`](../spec-python/), which is the base used for this adaptation.
`spec-.net/` was compared against it: `spec.md`, `data-model.md` and `api-contract.md` are identical in both versions.

## Precedence

1. [`../spec-python/constitution.md`](../spec-python/constitution.md) — non-negotiable principles.
2. [`../spec-python/spec.md`](../spec-python/spec.md), [`data-model.md`](../spec-python/data-model.md) and
   [`api-contract.md`](../spec-python/api-contract.md) — what is built. They win over anything in this folder.
3. This folder — how it is built with Laravel and React.

If something here contradicts the spec, this folder is wrong and gets fixed.

## Files

| File | Content |
|---|---|
| [`stack-mapping.md`](stack-mapping.md) | Each technical decision of the Python plan and its Laravel/React equivalent |
| [`spec-gaps.md`](spec-gaps.md) | Gaps and ambiguities found while reading the spec, and how they are handled without changing the business |
| [`tasks-laravel.md`](tasks-laravel.md) | Tasks T-01 to T-23 mapped to repositories and Laravel/React evidence |
