# Connection

Methods for establishing and inspecting the wallet ↔ dApp connection state.

## `connect`

Request a connection to the wallet. Triggers the approval popup on first call. Subsequent calls in the same browser session return the current state without re-prompting.

### Params

None.

### Returns

```ts
ConnectResult = {
  isConnected: boolean;       // true iff wallet is unlocked AND has a partyId
  reason: string;             // 'OK' | 'Wallet is locked' | 'No party onboarded' | ...
  isNetworkConnected: boolean;// true iff the active network's backend responded
  networkReason: string;
}
```

### Errors

- `4001 USER_REJECTED` — user declined the approval popup.

### Example

```ts
const result = await provider.request({ method: 'connect', params: {} });
if (!result.isConnected) {
  alert(`Wallet not ready: ${result.reason}`);
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
await provider.request({ method: 'disconnect', params: {} });
```

## `isConnected`

Same return shape as `connect` but never triggers an approval popup. Use for polling / health checks.

### Params

None.

### Returns

`ConnectResult` (same shape as `connect`).

### Example

```ts
const { isConnected } = await provider.request({ method: 'isConnected', params: {} });
```

## `status`

Richer health check than `isConnected`. Returns provider metadata, connection state, active network, and session info in one call.

### Params

None.

### Returns

```ts
StatusResult = {
  provider: { id: string; version: string; providerType: string };
  connection: { isConnected: boolean; reason: string };
  network: Network;            // { networkId, ledgerApi, name? }
  session: {
    isAuthenticated: boolean;  // wallet is unlocked
    partyId?: string;
  };
}
```

`provider.id` is always `'ginkgo'`. `provider.version` matches the extension manifest version. `provider.providerType` is `'browser'`.

### Errors

None.

### Example

```ts
const s = await provider.request({ method: 'status', params: {} });
console.log(`Connected to ${s.network.name ?? s.network.networkId} as ${s.session.partyId ?? '(locked)'}`);
```

This is the recommended method to call on window focus / page resume to detect changes (network switch, lock state) without a popup.

## `getActiveNetwork`

Returns just the active network. Useful when you don't need the rest of `status`.

### Params

None.

### Returns

```ts
Network = {
  networkId: string;   // 'localnet' | 'devnet' | 'testnet' | 'mainnet'
  ledgerApi: string;   // Backend base URL
  name?: string;       // Display label, e.g. 'Devnet'. Ginkgo extension; not in CIP-0103 spec.
}
```

### Errors

None.

### Example

```ts
const { networkId, ledgerApi } = await provider.request({ method: 'getActiveNetwork', params: {} });
console.log(`On ${networkId} via ${ledgerApi}`);
```

### Notes

- The user can change the active network via the Ginkgo popup at any time. dApps that hold per-network state should re-fetch this on focus, on `statusChanged` events (see [extensions](../extensions/ginkgo-vs-cip-0103.md)), or before each significant operation.
- There is no `setNetwork` method. Network selection is wallet-driven only.
- For Ethereum-style chain ID compatibility, treat `networkId` as the analog of `eth_chainId`'s return.
