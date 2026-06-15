# Prepare and execute

The transaction submission methods. These are the primary way dApps submit anything to the Canton ledger through Ginkgo. All three route through the wallet's connected backend, which mediates access to the underlying Canton ledger and enforces CIP-0103 allowlists on what can be submitted. For specifics of the backend Ginkgo ships against, see [appendix/ginkgo-backend.md](../appendix/ginkgo-backend.md).

## `prepareExecute`

Full transaction lifecycle: dApp submits a command, backend prepares a Canton transaction, user approves, wallet signs locally, backend executes on the ledger. Returns `null` per CIP-0103 — use [`prepareExecuteAndWait`](#prepareexecuteandwait) if you want the execute result.

### Params

```ts
PrepareExecuteParams = {
  commands: GatewayCommand[];           // Canton commands to execute
  userId?: string;                      // optional — defaults to active wallet user
  signingProviderId?: string;           // optional — defaults to 'ginkgo'
  applicationId?: string;               // optional — Canton application identifier
  meta?: Record<string, unknown>;       // optional — pass-through metadata
}

type GatewayCommand = {
  templateId: string;                   // e.g. '#Splice.Api.Token.TransferInstructionV1:TransferFactory'
  choiceName: string;                   // e.g. 'TransferFactory_Transfer'
  contractId?: string;                  // for exercise commands
  arguments: Record<string, unknown>;   // choice arguments
};
```

The exact `commands` shape depends on the Canton template package the backend is configured to allow. The backend enforces a strict `(templateId, choiceName)` allowlist — commands outside the allowlist return `-32003 TRANSACTION_REJECTED`. The default allowlist covers the Token Standard transfer + allocation flows; see [appendix/ginkgo-backend.md](../appendix/ginkgo-backend.md#token-standard-command-templates) for the full table. See the [send-tokens guide](../guides/send-tokens.md) for a concrete example.

### Returns

`null`.

Per the canonical CIP-0103 OpenRPC schema, `prepareExecute`'s result is `Null`. dApps that need transaction completion details should call `prepareExecuteAndWait` instead.

(Conceptually the spec also defines `txChanged` lifecycle events as a side-channel for `pending → signed → executed/failed` updates. Ginkgo doesn't emit these yet — see [extensions](../extensions/ginkgo-vs-cip-0103.md) for status.)

### Errors

- `4001 USER_REJECTED` — user declined the approval popup. Ginkgo also cleans up the pending command on the backend.
- `4100 UNAUTHORIZED` — wallet is locked or no party is onboarded.
- `-32002 RESOURCE_UNAVAILABLE` — backend is not configured for the active network.
- `-32003 TRANSACTION_REJECTED` — backend's Token Standard allowlist rejected the command, or the prepared transaction failed validation.
- `-32603 INTERNAL_ERROR` — backend returned an unexpected response, or no `commandId` could be extracted from the prepare reply.
- Other backend errors with their original codes (`-32000` to `-32005`) and messages are forwarded verbatim.

### Example

```ts
import { prepareExecute } from '@canton-network/dapp-sdk';

await prepareExecute({
  commands: [{
    templateId: '#Splice.Api.Token.TransferInstructionV1:TransferFactory',
    choiceName: 'TransferFactory_Transfer',
    arguments: { /* ... */ },
  }],
});
console.log('Transaction submitted'); // we don't get back the result; call prepareExecuteAndWait for that
```

## `prepareExecuteAndWait`

Same flow as `prepareExecute`, but returns the typed execute result wrapped in a CIP-0103 `TxChangedExecutedEvent`. Use this when you need access to the canonical `updateId` and `completionOffset`.

### Params

Same as `prepareExecute`.

### Returns

```ts
PrepareExecuteAndWaitResult = {
  tx: {
    status: 'executed';
    commandId: string;
    payload: {
      updateId: string;        // Canton ledger transaction ID
      completionOffset: number;// Ledger offset where the transaction landed
    };
  };
}
```

The outer `tx` wrapper matches the spec's `TxChangedExecutedEvent`. Don't strip it — strict-schema consumers of `@canton-network/dapp-sdk`'s `PrepareExecuteAndWaitResult` type expect the wrap.

### Errors

Same as `prepareExecute`.

### Example

```ts
import { prepareExecuteAndWait } from '@canton-network/dapp-sdk';

const result = await prepareExecuteAndWait({
  commands: [/* ... */],
});

console.log(`Transaction ${result.tx.commandId} executed`);
console.log(`Ledger updateId: ${result.tx.payload.updateId}`);
console.log(`Completion offset: ${result.tx.payload.completionOffset}`);
```

### Notes

- `commandId` is the backend-assigned identifier extracted from the prepare-step response. Stable across the prepare/sign/execute lifecycle.
- `updateId` is the Canton ledger transaction ID. Pass it to `ledgerApi` or your own Canton tooling to read the resulting events.
- `completionOffset` is the ledger offset where the transaction landed. Useful for resuming a stream of updates from this point forward.

## `ledgerApi`

Allowlisted proxy to a small subset of the Canton Ledger API, mediated by the wallet's backend. Used for read-only ledger queries. No approval popup — these are pull-style data fetches, not state changes.

**The backend enforces a strict allowlist on `(resource, requestMethod)` pairs.** Requests for anything outside the allowlist return `-32004 METHOD_NOT_SUPPORTED`. The CIP-0103 spec defines the method signature; the actual allowlist is a deployment decision of the backend Ginkgo ships against.

### Params

```ts
LedgerApiParams = {
  requestMethod: 'get' | 'post' | 'put' | 'delete';
  resource: string;        // path relative to the Canton Ledger API base
  body?: Record<string, unknown> | string;  // JSON object or stringified JSON
  query?: Record<string, string>;
  path?: Record<string, string>;
}
```

`requestMethod` is normalized to lowercase. `body` is normalized to an object (legacy callers passing stringified JSON are accepted).

### Returns

The backend's response payload. Shape depends entirely on the resource queried.

### Errors

- `4100 UNAUTHORIZED` — wallet is locked.
- `-32002 RESOURCE_UNAVAILABLE` — backend not configured for the active network, or the requested resource requires permissions the wallet user doesn't have (e.g., reading active contracts of a party they don't own).
- `-32001 RESOURCE_NOT_FOUND` — Canton Ledger API returned 404 for the resource.
- `-32004 METHOD_NOT_SUPPORTED` — the `(resource, requestMethod)` pair isn't in the backend's allowlist.
- Forwarded ledger errors with their original codes.

### What the allowlist looks like in practice

The reference backend that Ginkgo ships against currently allows three resources:

| Resource | Allowed methods | Notes |
|---|---|---|
| `/v2/version` | `GET` | Ledger API version metadata. |
| `/v2/state/ledger-end` | `GET` | Most recent ledger offset. |
| `/v2/state/active-contracts` | `POST` | Body must include `filter.filtersByParty` naming only parties the wallet user owns. `filtersForAnyParty` is rejected. |

Other resources return `-32004 METHOD_NOT_SUPPORTED`. See [appendix/ginkgo-backend.md](../appendix/ginkgo-backend.md#ledgerapi-resource-allowlist) for the up-to-date list.

### Example

Read an update by ID — note this is **not** currently in the default allowlist; the example illustrates the shape but would return `-32004` against the reference backend until the resource is whitelisted:

```ts
import { ledgerApi } from '@canton-network/dapp-sdk';

// Currently allowed: active contracts for a party the wallet user owns.
const account = await listAccounts().then((accs) => accs[0]);
const result = await ledgerApi({
  requestMethod: 'post',
  resource: '/v2/state/active-contracts',
  body: {
    filter: { filtersByParty: { [account.partyId]: {} } },
    verbose: true,
  },
});
```

### Notes

- This is the escape hatch for read-only Canton ledger queries that CIP-0103 doesn't otherwise expose. The backend's allowlist may grow over time as new resources are vetted; check the appendix or your deployment's docs for the current scope.
- All requests are mediated by the backend, which strips/transforms requests against its allowlist before reaching the underlying Canton Ledger API. The dApp does not authenticate itself; the wallet's session is what the backend sees.
- Restrictions on what counts as "your party" (for party-filtered resources) live on the backend side.
