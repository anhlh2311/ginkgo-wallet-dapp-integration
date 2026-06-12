# Prepare and execute

The transaction submission methods. These are the primary way dApps submit anything to the Canton ledger through Ginkgo. All three talk to the wallet's connected backend (the Wallet Gateway).

## `prepareExecute`

Full transaction lifecycle: dApp submits a command, gateway prepares a Canton transaction, user approves, wallet signs locally, gateway executes on the ledger.

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
  templateId: string;                   // e.g. 'Splice.Api.Token.TransferInstructionV1:TransferFactory'
  choiceName: string;                   // e.g. 'TransferFactory_Transfer'
  contractId?: string;                  // for exercise commands
  arguments: Record<string, unknown>;   // choice arguments
};
```

The exact `commands` shape depends on the Canton template package the gateway is configured to allow. See the [send-tokens guide](../guides/send-tokens.md) for a concrete example.

### Returns

The gateway's `execute` JSON-RPC result, typically:

```ts
{
  updateId: string;          // Canton ledger transaction ID
  completionOffset: number;  // ledger offset where the update was committed
  // ...possibly other gateway-defined fields
}
```

The exact shape is whatever the gateway returns and is not strictly typed by CIP-0103. For typed access to `updateId` + `completionOffset`, use [`prepareExecuteAndWait`](#prepareexecuteandwait) instead.

### Errors

- `4001 USER_REJECTED` — user declined the approval popup. Ginkgo also calls `deleteTransaction` on the gateway to clean up the pending command.
- `4100 UNAUTHORIZED` — wallet is locked or no party is onboarded.
- `-32002 RESOURCE_UNAVAILABLE` — wallet facade is not configured for the active network. Likely the network's `ledgerApi` URL is unreachable.
- `-32603 INTERNAL_ERROR` — gateway returned an unexpected response, or no `commandId` could be extracted from the prepare reply.
- Forwarded gateway errors with their original codes (`-32000` to `-32005`) and messages.

### Example

End-to-end token transfer via the Token Standard:

```ts
const command = {
  templateId: '#Splice.Api.Token.TransferInstructionV1:TransferFactory',
  choiceName: 'TransferFactory_Transfer',
  arguments: {
    sender: account.partyId,
    receiver: 'kairo-devnet::1220275036…30d',
    amount: '10.0',
    instrumentId: { admin: 'IDSO::1220be58…1a', id: 'Amulet' },
    // ... other Token Standard fields
  },
};

const result = await provider.request({
  method: 'prepareExecute',
  params: { commands: [command] },
});
console.log('Executed:', result);
```

The user sees an approval popup showing the prepared command details, then the transaction is signed locally and submitted. The promise resolves with the gateway's execute result.

See [Send a Token Standard transfer](../guides/send-tokens.md) for the full working example with all the Token Standard envelope fields populated.

## `prepareExecuteAndWait`

Same flow as `prepareExecute`, but returns the spec-typed `TxChangedExecutedEvent` shape — `{ status, commandId, payload: { updateId, completionOffset } }`. Use this when you need typed access to the canonical result fields.

### Params

Same as `prepareExecute`.

### Returns

```ts
{
  status: 'executed';
  commandId: string;
  payload: {
    updateId: string;
    completionOffset: number;
  };
}
```

### Errors

Same as `prepareExecute`.

### Example

```ts
const result = await provider.request({
  method: 'prepareExecuteAndWait',
  params: { commands: [command] },
});

console.log(`Transaction ${result.commandId} executed`);
console.log(`Ledger updateId: ${result.payload.updateId}`);
console.log(`Completion offset: ${result.payload.completionOffset}`);
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

Read the active updates stream:

```ts
const result = await provider.request({
  method: 'ledgerApi',
  params: {
    requestMethod: 'get',
    resource: '/v2/state/active-contracts',
    body: {
      filter: { filtersByParty: { [account.partyId]: {} } },
      verbose: true,
    },
  },
});
```

### Notes

- This is the escape hatch for things CIP-0103 doesn't otherwise expose. Use sparingly — the gateway's Ledger API surface may change without notice and isn't covered by the CIP-0103 stability guarantees.
- All requests are sent with the wallet user's bearer token attached. The dApp does not authenticate itself.
- Whatever restrictions exist on the resource (e.g., Token Standard choice allowlists) live on the gateway side, not in the wallet.
