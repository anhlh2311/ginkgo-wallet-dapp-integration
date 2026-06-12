# Prepare and execute

The transaction submission methods. These are the primary way dApps submit anything to the Canton ledger through Ginkgo. All three talk to the wallet's connected backend (the Wallet Gateway).

## `prepareExecute`

Full transaction lifecycle: dApp submits a command, gateway prepares a Canton transaction, user approves, wallet signs locally, gateway executes on the ledger. Returns `null` per CIP-0103 — use [`prepareExecuteAndWait`](#prepareexecuteandwait) if you want the execute result.

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

The exact `commands` shape depends on the Canton template package the gateway is configured to allow. See the [send-tokens guide](../guides/send-tokens.md) for a concrete example.

### Returns

`null`.

Per the canonical CIP-0103 OpenRPC schema, `prepareExecute`'s result is `Null`. dApps that need transaction completion details should call `prepareExecuteAndWait` instead.

(Conceptually the spec also defines `txChanged` lifecycle events as a side-channel for `pending → signed → executed/failed` updates. Ginkgo doesn't emit these yet — see [extensions](../extensions/ginkgo-vs-cip-0103.md) for status.)

### Errors

- `4001 USER_REJECTED` — user declined the approval popup. Ginkgo also calls `deleteTransaction` on the gateway to clean up the pending command.
- `4100 UNAUTHORIZED` — wallet is locked or no party is onboarded.
- `-32002 RESOURCE_UNAVAILABLE` — wallet facade is not configured for the active network.
- `-32603 INTERNAL_ERROR` — gateway returned an unexpected response, or no `commandId` could be extracted from the prepare reply.
- Forwarded gateway errors with their original codes (`-32000` to `-32005`) and messages.

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

- `commandId` is the gateway-assigned identifier extracted from the prepare-step response. Stable across the prepare/sign/execute lifecycle.
- `updateId` is the Canton ledger transaction ID. Pass it to `ledgerApi` or your own Canton tooling to read the resulting events.
- `completionOffset` is the ledger offset where the transaction landed. Useful for resuming a stream of updates from this point forward.

## `ledgerApi`

Generic proxy to the Wallet Gateway's Ledger API. Used for read-only ledger queries from a dApp. No approval popup — these are pull-style data fetches, not state changes.

### Params

```ts
LedgerApiParams = {
  requestMethod: 'get' | 'post' | 'put' | 'delete';
  resource: string;        // path relative to the gateway's Ledger API base
  body?: Record<string, unknown> | string;  // JSON object or stringified JSON
  query?: Record<string, string>;
  path?: Record<string, string>;
}
```

`requestMethod` is normalized to lowercase. `body` is normalized to an object (legacy callers passing stringified JSON are accepted).

### Returns

The gateway's response payload. Shape depends entirely on the resource queried.

### Errors

- `4100 UNAUTHORIZED` — wallet is locked.
- `-32002 RESOURCE_UNAVAILABLE` — wallet facade not configured for the active network.
- `-32001 RESOURCE_NOT_FOUND` — gateway returned 404 for the resource.
- Forwarded gateway errors.

### Example

Read an active update by ID:

```ts
import { ledgerApi } from '@canton-network/dapp-sdk';

const result = await ledgerApi({
  requestMethod: 'get',
  resource: '/v2/updates/update-by-id',
  body: { updateId: result.tx.payload.updateId },
});
```

### Notes

- This is the escape hatch for things CIP-0103 doesn't otherwise expose. Use sparingly — the gateway's Ledger API surface may change without notice and isn't covered by the CIP-0103 stability guarantees.
- All requests are sent with the wallet user's bearer token attached. The dApp does not authenticate itself.
- Whatever restrictions exist on the resource (e.g., Token Standard choice allowlists) live on the gateway side, not in the wallet.
