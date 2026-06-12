# Send a Token Standard transfer

End-to-end example: a dApp sends 10 Amulet tokens from the wallet user to another Canton party using the Canton [Token Standard](https://docs.canton.network/) `TransferFactory_Transfer` choice.

## What this guide covers

- Building a valid `TransferFactory_Transfer` command.
- Calling `prepareExecuteAndWait` so the wallet does the prepare → approve → sign → execute orchestration.
- Reading the resulting `updateId` and `completionOffset` to confirm the transaction landed on the ledger.

This is the canonical "submit a transaction" pattern. Anything more elaborate (multi-step workflows, choice arguments for non-Amulet tokens) starts here and varies only in the command shape.

## Prerequisites

- Ginkgo wallet installed, unlocked, with the user onboarded to a Canton party on the network of your choice.
- The Wallet Gateway running on the network's `ledgerApi` URL (Ginkgo connects to it automatically).
- The recipient's `partyId`. In the example below it's a placeholder — substitute a real party on the same network.
- The Token Standard's allowed choices configured on the gateway side. The wallet doesn't enforce this; the gateway will reject disallowed choices with a `-32003 TRANSACTION_REJECTED`.

## Complete example

```ts
import {
  init,
  connect,
  listAccounts,
  prepareExecuteAndWait,
  getConnectedProvider,
} from '@canton-network/dapp-sdk';

// 1. Initialize + connect.
await init();
await connect();

// 2. Identify the sender (active wallet user).
const accounts = await listAccounts();
const sender = accounts.find((a) => a.primary) ?? accounts[0];
if (!sender) throw new Error('Wallet has no primary account — sign in first');

const provider = getConnectedProvider();
const network = await provider!.request({ method: 'getActiveNetwork' });
console.log(`Sending from ${sender.partyId} on ${network.networkId}`);

// 3. Build the Token Standard transfer command.
//    Substitute real values for receiverPartyId and dsoPartyId — these are
//    placeholders illustrating the shape only.
const receiverPartyId = 'bob::1220abc...def0';
const dsoPartyId      = 'IDSO::1220beef...cafe';

const command = {
  templateId: '#Splice.Api.Token.TransferInstructionV1:TransferFactory',
  choiceName: 'TransferFactory_Transfer',
  arguments: {
    expectedAdmin: dsoPartyId,
    transfer: {
      sender: sender.partyId,
      receiver: receiverPartyId,
      amount: '10.0',
      instrumentId: {
        admin: dsoPartyId,
        id: 'Amulet',
      },
      requestedAt: new Date().toISOString(),
      executeBefore: new Date(Date.now() + 60 * 60_000).toISOString(),
      inputHoldingCids: [],   // gateway fills these in during prepare
      meta: {
        values: [
          { _1: 'splice.lfdecentralizedtrust.org/reason', _2: '' },
        ],
      },
    },
    extraArgs: {
      context: { values: [] },
      meta: { values: [] },
    },
  },
};

// 4. prepareExecuteAndWait does the whole prepare → approve → sign → execute dance.
//    The user sees an approval popup showing the prepared transaction details
//    and clicks Approve. The wallet then signs and submits.
const result = await prepareExecuteAndWait({ commands: [command] });

// 5. Inspect the result. Note the `tx` wrapper per CIP-0103.
console.log(`✅ Transfer executed`);
console.log(`  commandId: ${result.tx.commandId}`);
console.log(`  updateId: ${result.tx.payload.updateId}`);
console.log(`  completionOffset: ${result.tx.payload.completionOffset}`);
```

## What happens behind the scenes

1. **Prepare.** The wallet sends the command(s) to the gateway's `/api/v0/dapp` JSON-RPC `prepareExecute` method. The gateway builds a Canton transaction, computes the `preparedTransactionHash`, and returns a `{ userUrl }` containing a `commandId` query param.

2. **Approve.** The wallet pops the approval window in the user's browser, showing the prepared command details. The user clicks Approve (or Reject — if rejected, the wallet calls `deleteTransaction` to clean up the gateway's pending command and your `prepareExecuteAndWait` call rejects with `4001 USER_REJECTED`).

3. **Get transaction.** The wallet pulls the prepared transaction from `/api/v0/user` JSON-RPC `getTransaction { commandId }`. The response includes the `preparedTransactionHash` (base64-encoded 32-byte hash) the wallet needs to sign.

4. **Sign locally.** The wallet decrypts the user's private key from the in-memory cache and computes `signTransactionHash(preparedTransactionHash, privateKey)`. The signature is base64-encoded.

5. **Execute.** The wallet sends `/api/v0/user` JSON-RPC `execute { commandId, signature, signedBy: fingerprint, partyId }`. The gateway submits the signed transaction to the Canton participant node, which validates it, broadcasts to the network, and returns the `updateId` + `completionOffset`.

6. **Return to dApp.** The wallet wraps the result in CIP-0103's `TxChangedExecutedEvent` shape (`{ tx: { status: 'executed', commandId, payload: { updateId, completionOffset } } }`) and resolves your `prepareExecuteAndWait` promise.

Total round-trips: 3 to the gateway + 1 approval popup + 1 Ed25519 signing operation (~50ms total when the gateway is on the same machine).

## Error handling

```ts
try {
  const result = await prepareExecuteAndWait({ commands: [command] });
  // ...
} catch (e) {
  switch (e.code) {
    case 4001:
      // User rejected. Show neutral "transfer cancelled" UI, not an error.
      break;
    case 4100:
      // Wallet locked. Prompt user to unlock.
      break;
    case -32002:
      // Wallet facade not configured. The user's network is misconfigured.
      console.error('Wallet backend unreachable for active network.');
      break;
    case -32003:
      // Gateway rejected the prepared transaction. Usually a Token Standard
      // allowlist violation or insufficient holdings.
      console.error('Transaction rejected:', e.message);
      break;
    default:
      console.error('Unexpected error:', e);
  }
}
```

## Confirming the transfer landed

The promise resolves only after the gateway confirms the transaction was committed. By that point:

- The Canton ledger has the new transfer-offer / instruction contract created by the choice.
- The sender's amulet holdings are split / merged appropriately.
- The receiver can call their wallet's `transfer-offer/incoming-requests` endpoint to see the new offer (Ginkgo's own UI auto-refreshes this).

To independently verify, query the ledger with the returned `updateId`:

```ts
import { ledgerApi } from '@canton-network/dapp-sdk';

const ledgerResult = await ledgerApi({
  requestMethod: 'get',
  resource: '/v2/updates/update-by-id',
  body: { updateId: result.tx.payload.updateId },
});
console.log(ledgerResult);
```

## Tips

- **Use `prepareExecuteAndWait`, not `prepareExecute`,** when you need typed access to the `updateId`. The former returns the spec-shape `{ tx: TxChangedExecutedEvent }`; the latter returns `null` (per CIP-0103).
- **Don't construct `inputHoldingCids` yourself.** The gateway fills these during prepare. Pass an empty array; the gateway picks the right amulet contracts to consume based on the sender's holdings and the requested amount.
- **The `expectedAdmin` field must match the network's DSO party.** Ginkgo's gateway resolves this automatically when preparing, so the value you pass is validated against what the gateway computes. If they differ, the prepare step fails with `-32000 INVALID_INPUT` and a descriptive message.
- **Decimal amounts use string form.** `'10.0'`, not `10.0`. Daml's `Decimal` type doesn't round-trip through JavaScript number cleanly for large/precise values.
- **`executeBefore` is a wall-clock deadline.** If the gateway can't get the transaction committed before this time, it errors. Default to 1 hour for interactive flows; longer for batch.
