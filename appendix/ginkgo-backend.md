# Appendix: Ginkgo's backend implementation

> dApp developers integrating against the CIP-0103 surface **don't need to read this page**. It documents the specific backend service Ginkgo ships against, the JSON-RPC facade Ginkgo's background script talks to, and the allowlists that enforce CIP-0103 compliance. This is for support engineers, contributors, and anyone reverse-engineering an integration issue.

## Backend topology

The wallet's backend is a deployment-specific service that hides an upstream [Splice Wallet Gateway](https://github.com/hyperledger-labs/splice-wallet-kernel) behind a narrow, allowlisted JSON-RPC facade. The wallet **never speaks the Wallet Gateway protocol directly**, and the dApp **never sees the backend** — both layers are abstractions the wallet manages.

```
dApp
 └──► Ginkgo extension (background)
       └──► CIP-0103 facade on the wallet's backend
             ├── prepare/execute path → Splice Wallet Gateway (hidden) → Canton participant node
             └── ledgerApi path → Canton Ledger API (proxied through the allowlist)
```

The reference backend implementation that Ginkgo is built against is `canton-exchange-backend`. Other backends that expose the same facade contract are compatible.

## JSON-RPC facade endpoints

Two HTTP endpoints on the backend, both speaking JSON-RPC 2.0. The wallet sends requests; the backend dispatches to the appropriate handler.

| Endpoint | Auth | Purpose |
|---|---|---|
| `POST {ledgerApi}/api/v0/dapp` | None (the dApp's call context) | dApp-initiated methods: `prepareExecute`, `ledgerApi` |
| `POST {ledgerApi}/api/v0/user` | Wallet user's bearer token (`Authorization: Bearer <jwt>`) | User-authenticated follow-ups: `getTransaction`, `execute`, `deleteTransaction` |

`{ledgerApi}` is the base URL the wallet emits on `getActiveNetwork().ledgerApi`.

### Why two endpoints

- `/api/v0/dapp` is the unauthenticated entry point. A dApp request that hits Ginkgo's `prepareExecute` triggers an HTTP call from the wallet's background to this endpoint, carrying just the dApp-supplied command(s). The backend prepares a Canton transaction and returns a `userUrl` containing a `commandId` query parameter. The dApp is never told this URL — Ginkgo extracts the `commandId` and uses it internally.
- `/api/v0/user` is the authenticated follow-up. After the user approves in Ginkgo's popup, the wallet's background calls this endpoint (with the user's session bearer token) to fetch the prepared transaction (`getTransaction`), submit the signed transaction (`execute`), or clean up on rejection (`deleteTransaction`).

The two endpoints are how Ginkgo enforces the user-approval gate: the prepare step is cheap and unauthenticated; the execute step requires a signed, user-approved transaction.

## Allowlists

The backend enforces two allowlists. Requests outside them are rejected with `-32003 TRANSACTION_REJECTED` (Token Standard) or `-32004 METHOD_NOT_SUPPORTED` (resource).

### Token Standard command templates

`prepareExecute` commands are matched against `(templateId, choiceName)`. Only the entries in this table are accepted:

| Module:Template (interface) | Allowed choices |
|---|---|
| `Splice.Api.Token.TransferInstructionV1:TransferFactory` | `TransferFactory_Transfer` |
| `Splice.Api.Token.TransferInstructionV1:TransferInstruction` | `TransferInstruction_Accept`, `TransferInstruction_Reject`, `TransferInstruction_Withdraw` |
| `Splice.Api.Token.AllocationInstructionV1:AllocationFactory` | `AllocationFactory_Allocate` |
| `Splice.Api.Token.AllocationV1:Allocation` | `Allocation_Withdraw`, `Allocation_Cancel`, `Allocation_ExecuteTransfer` |

(Project-specific escrow templates may add additional entries in deployment configuration.)

A request that names any other template or choice fails with `-32003 TemplateNotAllowed` and the prepared transaction is not built.

### `ledgerApi` resource allowlist

`ledgerApi` is the escape hatch for read-only Canton ledger queries. The backend enforces a strict `(resource, requestMethod)` allowlist:

| Resource | Methods | Notes |
|---|---|---|
| `/v2/version` | `GET` | No body required. |
| `/v2/state/ledger-end` | `GET` | No body required. |
| `/v2/state/active-contracts` | `POST` | `body.filter.filtersByParty` is required and may only name parties the wallet user owns. `filtersForAnyParty` is rejected. Cross-party reads return `-32002 NotAuthorized`. |

Other resources or methods return `-32004 ResourceNotAllowed`.

This allowlist is the source of the `ledgerApi`'s narrow surface — the Canton Ledger API is much larger, but the facade exposes only this subset. Updates to the allowlist require a backend deployment.

## Auth model

- **dApp ↔ Ginkgo:** browser-extension postMessage. No HTTP, no credentials. See [concepts/auth-and-approval.md](../concepts/auth-and-approval.md).
- **Ginkgo ↔ backend `/api/v0/dapp`:** unauthenticated HTTP. The backend treats every prepare request as untrusted; allowlists run on every call.
- **Ginkgo ↔ backend `/api/v0/user`:** authenticated HTTP with the wallet user's `Authorization: Bearer <jwt>` header. The JWT is obtained via Google OAuth during onboarding and stored in the wallet's session storage.
- **Backend ↔ Wallet Gateway:** internal. Ginkgo doesn't see this path. The backend may use its own service-level credentials.
- **Backend ↔ Canton Ledger API:** internal. Same as above.

dApps never see the backend's bearer token. They never call the gateway directly. They never see the underlying Canton Ledger API. All of that is opaque, by design.

## Error code passthrough

When the backend's facade rejects a request, the JSON-RPC error code propagates verbatim back to the dApp:

- `-32003 TemplateNotAllowed` → dApp sees `TRANSACTION_REJECTED`
- `-32004 ResourceNotAllowed` → dApp sees `METHOD_NOT_SUPPORTED`
- `-32002 NotAuthorized` → dApp sees `RESOURCE_UNAVAILABLE`

These map cleanly to the CIP-0103 error code table in [reference/overview.md](../reference/overview.md#cip-0103-application-defined) — the facade doesn't invent codes, it uses the spec-defined application-error range.

## Why this matters for dApp developers

You don't need to do anything different in your dApp code as a result of this architecture — the CIP-0103 surface is the only thing you interact with. But knowing the allowlist exists explains:

- Why `prepareExecute` rejects exotic Daml templates with `-32003` even though the Canton ledger would accept them.
- Why `ledgerApi` can't fetch arbitrary endpoints from the Canton Ledger API — only the three resources above work today.
- Why some Canton-specific advanced operations (e.g., topology submission, identity management) aren't reachable through Ginkgo — they're outside the facade's allowlist.

For workflows that need surfaces beyond the allowlist, talk to the backend deployment team about whether they can be added, or use a non-wallet path (e.g., your own server-side Canton Ledger API client with its own credentials).
