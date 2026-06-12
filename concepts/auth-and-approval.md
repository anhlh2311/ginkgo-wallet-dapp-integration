# Authentication and approval

Ginkgo's trust model is straightforward but has a few subtleties worth understanding before you wire it up.

## Who is authenticated as whom

| Actor | Identity | How |
|---|---|---|
| **dApp page** | The page's origin (e.g., `https://your-dapp.example`) | Implicit — Ginkgo records the origin and uses it in approval popups. |
| **Wallet user** | A signed-in Canton party (`partyId`) | The user signed in to Ginkgo with Google OAuth + a password before any dApp call. |
| **Backend (Wallet Gateway)** | The wallet user's bearer token | Ginkgo holds an HS256 JWT in `sessionStore.authToken`, attaches it to every `/api/v0/user/*` call. |

**The dApp does not have its own credentials.** When you call `prepareExecute`, the wallet uses the user's bearer token to talk to the Gateway. From the Gateway's perspective, every dApp-initiated request is "the user is doing this." This is the same trust model as Ethereum wallet integrations.

This has two implications:

- **Server-side dApps cannot impersonate users.** They have to drive an *actual* browser session with Ginkgo installed and a logged-in user.
- **Phishing surface lives in the approval popup.** The user must look at the popup and decide whether to approve. The popup shows the dApp's origin, the method being called, and the request payload (for `signMessage`, the message text; for `prepareExecute`, the command list).

## Which methods require user approval

Methods in the `APPROVAL_REQUIRED_METHODS` set show an approval popup before the wallet does any work:

| Method | Approval popup? | What the user sees |
|---|---|---|
| `connect` | **Yes** | "Connect to <origin>?" Approve once per session. |
| `disconnect` | No (no-op) | — |
| `isConnected` / `status` | No | — |
| `getActiveNetwork` | No | — |
| `listAccounts` / `getPrimaryAccount` | No | — |
| `signMessage` | **Yes** | Message text + dApp origin + "Sign" button. |
| `signTransaction` | **Yes** | Hash to be signed + dApp origin. |
| `prepareExecute` / `prepareExecuteAndWait` | **Yes** | Prepared command details + dApp origin. |
| `ledgerApi` | No | — (used for read-only ledger queries) |

If the user clicks "Reject" (or closes the popup), Ginkgo returns a JSON-RPC error with code `4001 USER_REJECTED` and the dApp's request promise rejects.

```ts
try {
  await provider.request({ method: 'signMessage', params: { message: 'Hi' } });
} catch (e) {
  if (e.code === 4001) {
    // User rejected — show a non-error UI ("the user declined to sign").
    return;
  }
  throw e; // unexpected; surface as an error
}
```

## What "connected" means

`connect`'s return value is a tuple of two health bits:

```ts
{
  isConnected: boolean;       // wallet is unlocked AND has an onboarded party
  reason: string;             // 'OK' | 'Wallet is locked' | 'No party onboarded' | ...
  isNetworkConnected: boolean;// the wallet can reach the active network's API
  networkReason: string;      // 'OK' | error string
}
```

- `isConnected: true, isNetworkConnected: true` — green light, the user is signed in and the backend is reachable.
- `isConnected: false, reason: 'Wallet is locked'` — user unlocked once but the auto-lock timer fired (or they manually locked). They need to enter their password again.
- `isConnected: false, reason: 'No party onboarded'` — user signed in but hasn't completed Canton party onboarding. (Rare in practice — happens during first-run.)
- `isNetworkConnected: false` — backend HTTP errors. Ginkgo will surface the reason but the dApp can't fix it.

Treat the boolean conjunction `isConnected && isNetworkConnected` as the only "green" state. Anything else, prompt the user.

## Re-checking on resume

When the dApp regains focus after the user has been away, call `status` (no approval, no popup) to confirm the wallet's still in the same state:

```ts
window.addEventListener('focus', async () => {
  const s = await provider.request({ method: 'status', params: {} });
  if (!s.connection.isConnected) {
    showReconnectUI(s.connection.reason);
  } else if (s.network.networkId !== expectedNetworkId) {
    refreshDataForNetwork(s.network.networkId);
  }
});
```

This pattern is cheap (no backend round-trip) and avoids the wallet feeling stale.

## What's NOT authenticated

A dApp cannot:

- See what other dApps are connected.
- Call methods on behalf of any party other than the active one.
- Bypass the approval popup, even with prior approval — every sensitive call is reapproved unless the wallet's session is still warm from a recent approve (depending on the user's session settings; Ginkgo doesn't currently support "approve all from this origin" — every call shows a popup).

If a dApp wants persistent connection state, store the user's `partyId` and `networkId` locally after `connect`, and reconcile on each session.
