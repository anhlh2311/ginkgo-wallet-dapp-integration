# Overview

Ginkgo is a Canton Network wallet that lives in your browser as an extension. dApps integrate with it the same way they would with any EIP-1193-style wallet: through a `window.postMessage` channel speaking a defined JSON-RPC envelope. The envelope and method set are standardized as [CIP-0103](https://github.com/canton-foundation/cips/blob/main/cip-0103/cip-0103.md).

## What Ginkgo does for a dApp

- **Holds the user's Canton signing key** (Ed25519, base64-encoded). Encrypted at rest with the user's password (AES-GCM, PBKDF2-derived key).
- **Identifies the user as a Canton party** — `partyId` in the form `hint::fingerprint`, registered on a network.
- **Signs messages and transactions** the dApp asks for, after the user explicitly approves each one via popup.
- **Submits prepared transactions** to the Canton ledger via a Wallet Gateway backend, returning the final `updateId` and `completionOffset`.

## What CIP-0103 standardizes

CIP-0103 (Canton Improvement Proposal 103) defines the dApp-facing JSON-RPC API:

- 11 methods covering connection, account discovery, signing, and transaction lifecycle.
- A `SpliceMessage` envelope shape carrying requests/responses over `window.postMessage`.
- Discovery via [EIP-6963](https://eips.ethereum.org/EIPS/eip-6963)-style `canton:requestProvider` / `canton:announceProvider` `CustomEvent`s.
- A canonical set of JSON-RPC error codes (`-32700` to `-32603` reserved, `-32000` to `-32005` application-defined, plus EIP-1193 `4001` / `4100` / `4900` / `4901`).

The reference implementation lives at [hyperledger-labs/splice-wallet-kernel](https://github.com/hyperledger-labs/splice-wallet-kernel). The official SDK is `@canton-network/dapp-sdk`. Ginkgo implements the same wire protocol as the kernel and is compatible with the same SDK.

## What Ginkgo does NOT do

A dApp **cannot** use Ginkgo to:

- **Call the wallet's own private endpoints.** Ginkgo has its own backend for things like transfer-offer history, faucet, preapproval registration, and OAuth. Those are popup-only. dApps don't see them. (See [architecture.md](architecture.md) for why.)
- **Sign as a different user.** Whoever is signed into the Ginkgo wallet at the time of the call is the party that signs. The dApp does not provide its own credentials.
- **Switch the active network.** The user picks the network in the wallet popup. dApps can read it via `getActiveNetwork`, but there's no `setNetwork` method in CIP-0103.
- **Receive realtime push events from the wallet (in spec-conformant fashion).** This is a known gap in CIP-0103 for extension flows — see [extensions/ginkgo-vs-cip-0103.md](../extensions/ginkgo-vs-cip-0103.md).

## The 12-method dApp surface at a glance

| Method | Touches backend? | User approval popup? | Returns |
|---|---|---|---|
| `connect` | No | **Yes** | `ConnectResult` |
| `disconnect` | No | No | `null` |
| `isConnected` | No | No | `ConnectResult` |
| `status` | No | No | `StatusEvent` |
| `getActiveNetwork` | No | No | `Network` |
| `listAccounts` | No | No | `Wallet[]` |
| `getPrimaryAccount` | No | No | `Wallet` |
| `signMessage` | No (local Ed25519) | **Yes** | `{ signature }` |
| `signTransaction` (Ginkgo extension) | No (local Ed25519) | **Yes** | `{ signature, publicKey, fingerprint }` |
| `prepareExecute` | **Yes** (Wallet Gateway) | **Yes** (after gateway response) | `null` |
| `prepareExecuteAndWait` | **Yes** (Wallet Gateway) | **Yes** (after gateway response) | `{ tx: TxChangedExecutedEvent }` |
| `ledgerApi` | **Yes** (proxies to gateway) | No | gateway response |

Full reference: [reference/overview.md](../reference/overview.md).

## How to think about it

If you've integrated MetaMask or another EIP-1193 wallet, the mental model carries over almost directly:

- `eth_requestAccounts` → `connect` + `getPrimaryAccount`
- `personal_sign` → `signMessage`
- `eth_signTypedData` → no direct analog; closest is `signTransaction` (which signs a raw 32-byte hash, not typed data)
- `eth_sendTransaction` → `prepareExecute` (the wallet orchestrates prepare → sign → submit in one call)
- `eth_chainId` → `getActiveNetwork().networkId` (CAIP-2 form: `canton:<network>`)
- `wallet_switchEthereumChain` → no equivalent; users switch networks in the wallet UI
- `accountsChanged` / `chainChanged` events → see [extensions](../extensions/ginkgo-vs-cip-0103.md)

Canton's data model is more sophisticated than Ethereum's account model — there's the notion of a *party*, *Daml contracts*, *interactive submission*, and *external party amulet rules*. The wallet handles those details on your behalf via `prepareExecute`. You usually just submit a high-level command (e.g., "transfer 10 tokens from party X to party Y") and the wallet + Gateway turn it into a signable Canton transaction.
