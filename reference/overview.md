# Method index

All 12 dApp-callable JSON-RPC methods. Sortable at-a-glance reference; full details on the individual method pages.

| Method | Params | Returns | Approval popup? | Backend? | Spec? |
|---|---|---|---|---|---|
| [`connect`](connection.md#connect) | `{}` | `ConnectResult` | ✅ | — | CIP-0103 |
| [`disconnect`](connection.md#disconnect) | `{}` | `null` | — | — | CIP-0103 |
| [`isConnected`](connection.md#isconnected) | `{}` | `ConnectResult` | — | — | CIP-0103 |
| [`status`](connection.md#status) | `{}` | `StatusEvent` | — | — | CIP-0103 |
| [`getActiveNetwork`](connection.md#getactivenetwork) | `{}` | `Network` | — | — | CIP-0103 |
| [`listAccounts`](accounts.md#listaccounts) | `{}` | `Wallet[]` | — | — | CIP-0103 |
| [`getPrimaryAccount`](accounts.md#getprimaryaccount) | `{}` | `Wallet` | — | — | CIP-0103 |
| [`signMessage`](signing.md#signmessage) | `{ message }` | `{ signature }` | ✅ | — | CIP-0103 |
| [`signTransaction`](signing.md#signtransaction) | `{ transactionHash }` | `{ signature, publicKey, fingerprint }` | ✅ | — | Ginkgo extension |
| [`prepareExecute`](prepare-execute.md#prepareexecute) | `PrepareExecuteParams` | `null` | ✅ | ✅ | CIP-0103 |
| [`prepareExecuteAndWait`](prepare-execute.md#prepareexecuteandwait) | `PrepareExecuteParams` | `{ tx: TxChangedExecutedEvent }` | ✅ | ✅ | CIP-0103 |
| [`ledgerApi`](prepare-execute.md#ledgerapi) | `LedgerApiParams` | gateway response | — | ✅ | CIP-0103 |

## Common shapes

Used across multiple methods. Defined once here.

### `JsonRpcResponse`

```ts
type JsonRpcResponse =
  | { jsonrpc: '2.0'; id: string | number | null; result: unknown }
  | { jsonrpc: '2.0'; id: string | number | null; error: { code: number; message: string; data?: unknown } };
```

### `Network`

```ts
interface Network {
  networkId: string;   // CAIP-2-compliant chain ID, e.g. 'canton:localnet', 'canton:devnet'
  ledgerApi?: string;  // Base URL of the network's Wallet Gateway / Ledger API
  accessToken?: string;// Optional bearer token; Ginkgo never emits this field
}
```

`additionalProperties: false` — strict OpenRPC validators reject any extra fields.

### `Wallet`

The `DappAccount` shape returned by `listAccounts` and `getPrimaryAccount`. Mirrors the CIP-0103 `Wallet` schema.

```ts
interface Wallet {
  primary: boolean;                                    // true for the active wallet
  partyId: string;                                     // 'hint::fingerprint'
  status: 'initialized' | 'allocated' | 'removed';    // currently always 'allocated' in Ginkgo
  hint: string;                                        // pre-'::' portion of partyId
  publicKey: string;                                   // base64-encoded Ed25519 public key
  namespace: string;                                   // post-'::' portion (the fingerprint)
  networkId: string;                                   // CAIP-2 form, matches getActiveNetwork().networkId
  signingProviderId: string;                           // always 'ginkgo'
  externalTxId?: string;
  topologyTransactions?: string;
  disabled?: boolean;
  reason?: string;
}
```

### `ConnectResult`

```ts
interface ConnectResult {
  isConnected: boolean;        // wallet unlocked AND has an onboarded party
  reason?: string;             // 'OK' or a human-readable reason (Ginkgo always populates)
  isNetworkConnected: boolean; // wallet can reach the active network's backend
  networkReason?: string;      // 'OK' or a human-readable reason (Ginkgo always populates)
  userUrl?: string;            // Optional async-connect URL; Ginkgo never emits (sync-only)
}
```

### `Session`

```ts
interface Session {
  accessToken: string;  // JWT bearer token from the wallet's auth session
  userId: string;       // Stable user identifier from the OAuth session
}
```

`additionalProperties: false`. Ginkgo emits this only when both fields are available; otherwise the optional `status.session` field is omitted entirely.

### `StatusEvent`

```ts
interface StatusEvent {
  provider: {
    id: string;          // 'ginkgo'
    version: string;     // matches manifest version, e.g. '0.2.0'
    providerType: 'browser' | 'desktop' | 'mobile' | 'remote';  // always 'browser' for Ginkgo
  };
  connection: ConnectResult;  // same shape as connect()
  network?: Network;          // optional per spec; Ginkgo emits when a network is configured
  session?: Session;          // optional per spec; Ginkgo emits when signed in
}
```

## Error codes

All methods can return JSON-RPC errors with these codes. Per CIP-0103.

### Standard JSON-RPC

| Code | Name | When |
|---|---|---|
| `-32700` | `PARSE_ERROR` | Invalid JSON sent. |
| `-32600` | `INVALID_REQUEST` | Not a valid JSON-RPC request. |
| `-32601` | `METHOD_NOT_FOUND` | Method name not in the dispatch table. |
| `-32602` | `INVALID_PARAMS` | Params missing or malformed (e.g., `message` is a number, `transactionHash` is hex). |
| `-32603` | `INTERNAL_ERROR` | Unexpected throw. Catch-all. |

### CIP-0103 application-defined

| Code | Name | When |
|---|---|---|
| `-32000` | `INVALID_INPUT` | Generic input validation failure. |
| `-32001` | `RESOURCE_NOT_FOUND` | Requested entity doesn't exist on the backend. |
| `-32002` | `RESOURCE_UNAVAILABLE` | Backend reachable but resource is currently unavailable (e.g., wallet facade not configured for this network). |
| `-32003` | `TRANSACTION_REJECTED` | Backend rejected the prepared transaction. |
| `-32004` | `METHOD_NOT_SUPPORTED` | Method exists on the spec but Ginkgo doesn't implement it. |
| `-32005` | `LIMIT_EXCEEDED` | Rate or size limit hit. |

### EIP-1193 provider

| Code | Name | When |
|---|---|---|
| `4001` | `USER_REJECTED` | User clicked Reject in an approval popup, or closed it. |
| `4100` | `UNAUTHORIZED` | Wallet is locked, or no party is onboarded. |
| `4200` | `UNSUPPORTED_METHOD` | (Reserved; Ginkgo currently uses `-32601` for unknown methods.) |
| `4900` | `DISCONNECTED` | Wallet is offline / unreachable. |
| `4901` | `CHAIN_DISCONNECTED` | Network backend is unreachable. |

All error responses follow the JSON-RPC envelope:

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "error": { "code": 4001, "message": "User rejected the request" }
}
```
