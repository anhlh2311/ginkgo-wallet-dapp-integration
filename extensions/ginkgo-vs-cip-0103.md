# Ginkgo vs CIP-0103

A precise list of where Ginkgo's dApp API deviates from vanilla CIP-0103. Every deviation is intentional and documented here so dApp developers know what to expect.

## Categories

- **Extensions** — Ginkgo provides more than the spec defines. dApps using only the spec aren't affected.
- **Limitations** — Ginkgo provides less than the spec defines. Documented so dApps don't expect the missing surface.
- **Deferred upstream** — gaps the spec defines but the official `@canton-network/dapp-sdk` doesn't implement either, awaiting upstream wiring.

## Extensions

### `signTransaction` method

CIP-0103 has no `signTransaction` method. Ginkgo adds one to sign a 32-byte transaction hash directly. See [reference/signing.md](../reference/signing.md#signtransaction).

**Why kept:** existing Canton dApps (notably `canton-test-dapp`) rely on it. For new code, prefer `prepareExecute` — it follows the spec.

**Return shape:** `{ signature, publicKey, fingerprint }` (extension shape; spec has no equivalent).

### `name` field on `Network`

CIP-0103's `Network` shape defines `networkId` (required) and `ledgerApi` (optional). Ginkgo's response also includes `name`, a human-readable label like `'Localnet'` / `'Devnet'`.

**Why kept:** UI display convenience. CIP-0103 permits `additionalProperties`, so schema-validating SDKs accept the extra field. Spec-strict consumers can ignore it.

### `SPLICE_WALLET_EVENT` envelope variant

The standard `WalletEvent` enum has 7 members covering request/response and the extension handshake. Ginkgo adds an 8th: `SPLICE_WALLET_EVENT`, used to push `statusChanged` / `accountsChanged` notifications from the background to the dApp.

```ts
// Ginkgo-pushed event envelope:
{
  type: 'SPLICE_WALLET_EVENT',
  event: 'statusChanged' | 'accountsChanged',
  data: unknown,  // for statusChanged: StatusResult; for accountsChanged: DappAccount[]
}
```

**Why kept:** see [Deferred upstream](#deferred-upstream) below — upstream's `DappSyncProvider` (the extension flow) doesn't yet have any event delivery mechanism, so even spec-conformant push events have no consumer. Ginkgo's own internal logic and downstream Ginkgo-aware dApps use this channel today.

**How a dApp consumes it (without the SDK):**

```js
window.addEventListener('message', (e) => {
  const m = e.data;
  if (m?.type !== 'SPLICE_WALLET_EVENT') return;
  switch (m.event) {
    case 'statusChanged':
      refreshUiFor(m.data);  // m.data has the StatusResult shape
      break;
    case 'accountsChanged':
      onAccountsChanged(m.data);  // m.data is a DappAccount[]
      break;
  }
});
```

When upstream ships a spec-conformant push channel for the extension flow, this envelope will be deprecated in favor of the upstream mechanism. Until then, Ginkgo offers it as the only way to receive realtime wallet state updates.

## Limitations

### Single-account wallet

CIP-0103's `listAccounts` is defined to return an array of all accounts the wallet manages. Ginkgo currently returns at most one account — the active party for the current network.

**Why:** the wallet's onboarding flow creates exactly one party per network per user. Multi-party-per-user isn't a product requirement yet.

**Impact:** dApps should still iterate `listAccounts` rather than assume length 1, in case Ginkgo gains multi-account support in the future.

### `connect` is synchronous-only (no IdP login)

CIP-0103 defines an async `connect` variant where the wallet returns a `{ userUrl }` and the dApp drives the user through an IdP (Google OAuth, etc.) login flow before connection completes.

Ginkgo's `connect` is always synchronous. The user has already signed into Google OAuth via the wallet popup *before* any dApp call; the dApp never sees the auth flow.

**Impact:** no `SPLICE_WALLET_IDP_AUTH_SUCCESS` event will ever be fired from Ginkgo. The envelope variant is accepted (for compatibility) but never originated. If a dApp expects to drive the login flow, it won't work with Ginkgo today.

### No `messageSignature` lifecycle events

CIP-0103's spec defines `messageSignature` events (`pending` → `signed`/`failed`) that fire during a `signMessage` call. Ginkgo doesn't emit them.

**Why:** Ginkgo's `signMessage` is synchronous behind the approval popup — the user clicks Approve, the wallet signs immediately, the promise resolves. There's no "pending" phase visible to the dApp that would benefit from a lifecycle event.

**Impact:** dApps that subscribe to `messageSignature` events won't receive any from Ginkgo. Use the promise return from `signMessage` instead.

### `connect` return when already connected

When the user has connected once and calls `connect` again in the same session, Ginkgo returns the current `ConnectResult` rather than re-prompting for approval. CIP-0103 doesn't explicitly mandate either behavior.

**Impact:** dApps can call `connect` idempotently to check connection state, but should still use `isConnected` or `status` for explicit health checks since they're guaranteed not to pop up the approval window.

## Deferred upstream

### `prepareExecute` lifecycle events

CIP-0103 defines `txChanged` lifecycle events (`pending` → `signed` → `executed`/`failed`) that fire during a `prepareExecute` call so the dApp can show progress UI without polling.

**Status in Ginkgo:** not emitted. Our `prepareExecute` is "fire and resolve" — the promise resolves after the gateway returns the execute result.

**Status upstream:** the `@canton-network/dapp-sdk`'s `DappSyncProvider` (used for extension flows) uses `WindowTransport`, which is request/response only. It has no listener for these events. The reference wallet-gateway extension stubs `txChanged` with `throw new Error('Only for events.')`. So even if Ginkgo emitted spec-conformant events, no SDK consumer would receive them.

**When this will be fixed:** awaiting upstream wiring of an event delivery channel for the extension flow. Track at [hyperledger-labs/splice-wallet-kernel](https://github.com/hyperledger-labs/splice-wallet-kernel).

In the meantime, dApps can use:
- The promise return from `prepareExecuteAndWait` (which carries the spec-shape executed event).
- Ginkgo's `SPLICE_WALLET_EVENT` channel for `statusChanged` / `accountsChanged` (Ginkgo extension, see above).

### Multi-wallet routing (`target` field)

CIP-0103 defines an optional `target: string` on the SpliceMessage envelope so dApps can route requests to a specific wallet extension by ID when multiple are installed.

**Status in Ginkgo:** supported on incoming (request/ext-ready/ext-open) and echoed on outgoing (ext-ack). dApps using `@canton-network/dapp-sdk`'s discovery client will see Ginkgo announced with `target: chrome.runtime.id` and can route subsequent calls correctly.

**Note:** if a dApp omits `target`, Ginkgo accepts the message and processes it — same behavior as upstream.

## Quick reference: divergence summary table

| Feature | CIP-0103 | Ginkgo |
|---|---|---|
| Method set | 11 methods | 12 methods (+`signTransaction`) |
| `signMessage` returns | `{ signature }` | `{ signature }` (spec-compliant) |
| `Network` shape | `{ networkId, ledgerApi?, accessToken? }` | `{ networkId, ledgerApi, name }` (+`name`) |
| `WalletEvent` envelope members | 7 (REQUEST, RESPONSE, EXT_*, IDP_AUTH_SUCCESS, LOGOUT) | 8 (+`SPLICE_WALLET_EVENT`) |
| `target` field on envelope | optional | supported, echoed |
| Discovery | EIP-6963 (`canton:requestProvider` / `canton:announceProvider`) | EIP-6963 (spec-compliant) |
| `txChanged` lifecycle events | defined | not emitted (deferred upstream) |
| `accountsChanged` / `statusChanged` push | spec-defined channel TBD | emitted via `SPLICE_WALLET_EVENT` (Ginkgo channel) |
| Async `connect` + IdP login | defined | not implemented (Ginkgo holds keys locally) |
| Multi-account | defined | single-account today |
| Error codes | full set | full set (PR 6) |
