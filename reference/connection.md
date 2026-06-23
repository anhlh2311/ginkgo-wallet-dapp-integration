# Connection

Methods for establishing and inspecting the wallet ↔ dApp connection state.

## `connect`

Request a connection to the wallet. Triggers the approval popup on first call. Subsequent calls in the same browser session return the current state without re-prompting.

### Params

None.

### Returns

```ts
ConnectResult = {
  isConnected: boolean;        // true iff wallet is unlocked AND has a partyId
  reason?: string;             // 'OK' | 'Wallet is locked' | 'No party onboarded' | ...
  isNetworkConnected: boolean; // true iff the active network's backend responded
  networkReason?: string;
}
```

Spec required: `isConnected`, `isNetworkConnected`. Ginkgo always populates `reason` and `networkReason` for additional context, but spec-validating consumers should treat them as optional.

### Errors

- `4001 USER_REJECTED` — user declined the approval popup.

### Example

```ts
import { connect } from '@canton-network/dapp-sdk';

const result = await connect();
if (!result.isConnected) {
  alert(`Wallet not ready: ${result.reason ?? 'unknown'}`);
}
```

## `disconnect`

Acknowledges the dApp's intent to disconnect. Currently a no-op in Ginkgo — the wallet doesn't track per-dApp connection state. Safe to call on logout.

### Params

None.

### Returns

`null`.

### Example

```ts
import { disconnect } from '@canton-network/dapp-sdk';

await disconnect();
```

## `isConnected`

Same return shape as `connect` but never triggers an approval popup. Use for polling / health checks.

### Params

None.

### Returns

`ConnectResult` (same shape as `connect`).

### Example

```ts
import { isConnected } from '@canton-network/dapp-sdk';

const { isConnected: ok } = await isConnected();
```

## `status`

Richer health check than `isConnected`. Returns provider metadata, connection state, active network, and session info in one call.

### Params

None.

### Returns

```ts
StatusEvent = {
  provider: {
    id: string;          // 'ginkgo'
    version: string;     // matches manifest version
    providerType: 'browser' | 'desktop' | 'mobile' | 'remote';  // 'browser' for Ginkgo
  };
  connection: ConnectResult;  // same shape as connect()
  network?: Network;          // optional per spec; Ginkgo emits when configured
  session?: Session;          // optional per spec; Ginkgo emits when signed in
}

Session = {
  accessToken: string;  // bearer token from the wallet's OAuth session
  userId: string;       // Stable user identifier from the OAuth session
}
```

Notes:
- `provider.id` is `'ginkgo'`. `provider.version` matches the extension manifest. `provider.providerType` is `'browser'`.
- `status.session` follows the spec's `Session` shape (`accessToken` + `userId`) and is omitted entirely if the user isn't signed in. **It does not echo `partyId`** — use `listAccounts()` or `getPrimaryAccount` for party-level identity.
- `status.network` follows the spec's `Network` shape (`networkId` + optional `ledgerApi`/`accessToken`) and is omitted entirely if no network is configured.

### Errors

None.

### Example

```ts
import { status } from '@canton-network/dapp-sdk';

const s = await status();
console.log(`Connected to ${s.network?.networkId ?? '(no network)'}`);
if (s.session) {
  console.log(`Authenticated as user ${s.session.userId}`);
}
```

This is the recommended method to call on window focus / page resume to detect changes (network switch, lock state) without a popup.

## `getActiveNetwork`

Returns just the active network. Useful when you don't need the rest of `status`.

### Params

None.

### Returns

```ts
Network = {
  networkId: string;   // CAIP-2-compliant chain ID, e.g. 'canton:localnet', 'canton:devnet'
  ledgerApi?: string;  // Backend base URL
  accessToken?: string;// Optional bearer token; Ginkgo never emits this field
}
```

`additionalProperties: false`. The shape contains exactly these three fields; strict OpenRPC validators reject any extras.

### Errors

None.

### Example

```ts
import { getConnectedProvider } from '@canton-network/dapp-sdk';

const provider = getConnectedProvider();
if (!provider) throw new Error('Not connected');
const { networkId, ledgerApi } = await provider.request({ method: 'getActiveNetwork' });
console.log(`On ${networkId} via ${ledgerApi}`);
```

### Notes

- `networkId` is in [CAIP-2](https://chainagnostic.org/CAIPs/caip-2) form (`canton:<network>`). Ginkgo emits values like `'canton:localnet'`, `'canton:devnet'`, `'canton:testnet'`, `'canton:mainnet'`.
- The user can change the active network via the Ginkgo popup at any time. dApps that hold per-network state should re-fetch this on focus, on `statusChanged` events (see [extensions](../extensions/ginkgo-vs-cip-0103.md)), or before each significant operation.
- There is no `setNetwork` method. Network selection is wallet-driven only.
- For an EVM-style chain ID analog, treat `networkId` as the equivalent of `eth_chainId`'s return — both serve to disambiguate which chain a wallet is signing against.
- `getActiveNetwork` isn't exposed as a top-level SDK helper (yet); reach it through `getConnectedProvider().request({ method: 'getActiveNetwork' })`. Most dApps can read the same information from `status().network`.
