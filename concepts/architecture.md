# Architecture

When a dApp calls a Ginkgo method, the request travels through four contexts before any work happens. Knowing this map saves hours of debugging when something goes wrong.

## Message flow

```
   ┌─────────────────────┐
   │ dApp page (your JS) │
   └──────────┬──────────┘
              │ window.postMessage({ type: 'SPLICE_WALLET_REQUEST', request: ... })
              ▼
   ┌─────────────────────────────────────────┐
   │ Content script (entrypoints/content.ts) │  Runs on every URL at document_start.
   │ - shouldHandle(target) check            │  Routes to background; relays response back.
   └──────────┬──────────────────────────────┘
              │ chrome.runtime.sendMessage(msg)
              ▼
   ┌──────────────────────────────────────────────────────┐
   │ Background service worker                            │  Manages session, keystore, network.
   │ - dapp-api.handler.ts dispatcher                     │  Pops approval popup for sensitive methods.
   │ - methods table → handleSignMessage, handleConnect, …│
   └──────────┬───────────────────────────────────────────┘
              │ For prepareExecute / ledgerApi:
              ▼
   ┌────────────────────────────────────────────────────────────┐
   │ Wallet Gateway (the wallet's connected backend)            │  REST + JSON-RPC facade.
   │ POST /api/v0/dapp { method, params }  (dApp-context calls) │  /api/v0/dapp = unauthenticated
   │ POST /api/v0/user { method, params }  (user-auth'd calls)  │  /api/v0/user = wallet user bearer
   └────────────────────────────────────────────────────────────┘
              │
              ▼
        Canton ledger
```

Returns travel back along the same path: gateway → background → content script → `window.postMessage({ type: 'SPLICE_WALLET_RESPONSE', response: { ... } })` → dApp's `message` listener.

## The four contexts

### 1. dApp page

Your code. Runs in the page's normal JavaScript context with the page's origin. No special permissions. Talks to Ginkgo *only* through `window.postMessage` (or the SDK wrapping it).

### 2. Content script

Lives at `entrypoints/content.ts` in the wallet's source. Runs in an *isolated* world — it shares the DOM with your page but has its own JS heap. The bridge between page-context `window.postMessage` and extension-context `chrome.runtime.sendMessage`. Also handles:

- **Provider discovery** — listens for `canton:requestProvider`, dispatches `canton:announceProvider` with Ginkgo's identity.
- **Target routing** — checks `msg.target` against `chrome.runtime.id`; ignores messages addressed to a different wallet.
- **Same-window filtering** — drops messages whose `event.source !== window` to prevent iframe injection.

### 3. Background service worker

MV3 service worker at `entrypoints/background.ts`. Holds the user's session state (`partyId`, `authToken`, cached unlocked private key), encrypted keystore, current network. Dispatches JSON-RPC methods to handlers in `entrypoints/background/handlers/`. Lifecycle: may sleep when idle, wakes on a `chrome.runtime` message.

This is where:

- Approval popups are triggered (via `chrome.windows.create`).
- The private key is decrypted (only while the wallet is unlocked, held in a JS variable that dies when the service worker sleeps).
- All `signMessage` / `signTransactionHash` calls happen.
- Gateway HTTP requests are issued (`gatewayFacadeDappRpc` / `gatewayFacadeUserRpc`).

### 4. Wallet Gateway (Canton-exchange backend)

A separate service the wallet talks to. On Ginkgo's dev/devnet/testnet/mainnet setups it's at `http://localhost:3003` / `https://api-devnet.kairo.ag` / etc. Implements two JSON-RPC facades:

- `POST /api/v0/dapp` — unauthenticated. Used for `prepareExecute` (taking a dApp-issued command and turning it into a prepared transaction the user can review).
- `POST /api/v0/user` — authenticated with the *wallet user's* bearer token. Used for `getTransaction`, `execute`, `deleteTransaction`. **This is the auth boundary** — dApps don't authenticate themselves; they piggyback on the active wallet user's session.

The Gateway in turn talks to a Canton participant node, signing relay, and ledger API.

## Why dApps can't see the wallet's private endpoints

The wallet's own backend has endpoints like `/transfer-offer/history`, `/wallet/transfer-preapproval/status`, `/faucet/*`, `/auth/me`. These are wallet UI features — they appear in the popup's tabs. **They're not exposed to dApps** because:

1. They'd let any visited website read the user's full transfer history and active session metadata. That's a privacy leak the spec deliberately avoids.
2. The CIP-0103 surface is intentionally narrow — connect, sign, prepare, execute. Anything else lives behind a different abstraction.

If a dApp wants something more than the spec offers, it should call the Wallet Gateway directly with the user's permission (e.g., via `ledgerApi` for raw Canton Ledger API access).

## Lifecycle notes

- **Service worker sleep:** the background may idle out between calls. The first call after a sleep cycle takes ~50-200ms longer than subsequent calls in the same wake window. This is normal MV3 behavior.
- **Locked wallet:** if the user has locked Ginkgo (or the auto-lock timer has fired), the cached private key is gone. `connect` returns `isConnected: false` with `reason: 'Wallet is locked'`. dApps should treat this as a transient state and prompt the user to unlock.
- **Popup approval blocking:** when the user has an approval popup open, the dApp call is suspended awaiting their decision. Calls can wait minutes. Plan for this — the SDK request returns a promise; don't expect <100ms latency on `signMessage` or `prepareExecute`.
- **Network change:** the user can switch networks in the popup at any time. Ginkgo emits a `statusChanged` event via `SPLICE_WALLET_EVENT` (see [extensions](../extensions/ginkgo-vs-cip-0103.md)). dApps that hold state should re-fetch `getActiveNetwork` and `getPrimaryAccount` on resume.
