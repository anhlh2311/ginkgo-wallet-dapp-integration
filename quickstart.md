# Quickstart

Connect to Ginkgo, sign a message, verify the signature. 5 minutes, no backend required.

## Prerequisites

- Ginkgo wallet installed and unlocked in your browser ([install guide](https://github.com/anhlh2311/ginkgo#install)).
- A development dApp page running on any URL. The wallet's content script matches `<all_urls>`, so it works on `localhost`, `https://example.com`, or anywhere.
- Node 20+ if you want to use the SDK; otherwise the raw protocol works from any browser DevTools console.

## Option A — Using `@canton-network/dapp-sdk` (recommended)

```bash
npm install @canton-network/dapp-sdk tweetnacl tweetnacl-util
```

```ts
import { DiscoveryClient } from '@canton-network/dapp-sdk';
import nacl from 'tweetnacl';
import naclUtil from 'tweetnacl-util';

// 1. Discover the wallet via EIP-6963-style announce.
const providers = await new DiscoveryClient().listProviders({ timeoutMs: 300 });
const ginkgo = providers.find((p) => p.name === 'Ginkgo');
if (!ginkgo) throw new Error('Ginkgo wallet not detected — is it installed and unlocked?');

// 2. Connect.
const provider = await new DiscoveryClient().connect(ginkgo);
const status = await provider.request({ method: 'connect', params: {} });
console.log('Connected:', status);

// 3. Cache the publicKey once. signMessage returns { signature } only.
const account = await provider.request({ method: 'getPrimaryAccount', params: {} });
const cachedPublicKey = account.publicKey;

// 4. Sign a message. The user will see an approval popup.
const message = 'Hello from my dApp';
const { signature } = await provider.request({
  method: 'signMessage',
  params: { message },
});

// 5. Verify locally with any standard Ed25519 verifier.
const verified = nacl.sign.detached.verify(
  new TextEncoder().encode(message),
  naclUtil.decodeBase64(signature),
  naclUtil.decodeBase64(cachedPublicKey),
);
console.log(verified ? '✅ verified' : '❌ verification failed');
```

## Option B — Raw `window.postMessage`

Same flow, no SDK dependency. Paste into any page's DevTools console:

```js
// Minimal JSON-RPC client over postMessage.
let reqId = 1;
function rpc(method, params) {
  const id = reqId++;
  return new Promise((resolve, reject) => {
    function onMsg(e) {
      const m = e.data;
      if (m?.type !== 'SPLICE_WALLET_RESPONSE') return;
      if (m.response?.id !== id) return;
      window.removeEventListener('message', onMsg);
      if (m.response.error) reject(m.response.error);
      else resolve(m.response.result);
    }
    window.addEventListener('message', onMsg);
    window.postMessage(
      { type: 'SPLICE_WALLET_REQUEST', request: { jsonrpc: '2.0', id, method, params } },
      '*',
    );
  });
}

// Use it.
await rpc('connect', {});
const account = await rpc('getPrimaryAccount', {});
const { signature } = await rpc('signMessage', { message: 'Hello' });
console.log({ account, signature });
```

To verify the signature, load `tweetnacl` from a CDN and follow Option A's step 5:

```js
const [naclMod, naclUtilMod] = await Promise.all([
  import('https://esm.sh/tweetnacl@1.0.3'),
  import('https://esm.sh/tweetnacl-util@0.15.1'),
]);
const nacl = naclMod.default || naclMod;
const naclUtil = naclUtilMod.default || naclUtilMod;

const verified = nacl.sign.detached.verify(
  new TextEncoder().encode('Hello'),
  naclUtil.decodeBase64(signature),
  naclUtil.decodeBase64(account.publicKey),
);
console.log(verified ? '✅' : '❌');
```

## What just happened

1. The dApp dispatched a `canton:requestProvider` `CustomEvent`. Ginkgo's content script (running on every page at `document_start`) responded with `canton:announceProvider` carrying the wallet's identity. The SDK collected this into the discovery result.

2. The dApp called `connect`. Ginkgo's content script forwarded the JSON-RPC request to the background service worker, which checked unlock state and returned a `{ isConnected, reason, isNetworkConnected, networkReason }` object.

3. The dApp called `getPrimaryAccount`. Ginkgo returned the `DappAccount` for the currently active wallet: `{ partyId, publicKey, hint, networkId, ... }`.

4. The dApp called `signMessage`. Because `signMessage` is in the `APPROVAL_REQUIRED_METHODS` set, the user saw a popup with the message preview and approved. The wallet computed `nacl.sign.detached(utf8(message), privateKey)`, base64-encoded the 64-byte signature, and returned `{ signature }` — and only `{ signature }`, per CIP-0103.

5. Locally, the dApp verified the signature against the publicKey cached from step 3.

## Next steps

- [Method reference](reference/overview.md) — every method, signature, return shape, and error codes.
- [Send a Token Standard transfer](guides/send-tokens.md) — end-to-end transaction flow using `prepareExecute`.
- [Verify a signature](guides/verify-signatures.md) — verifier code for the three common stacks (`tweetnacl`, `libsodium`, Node `crypto.subtle`).
- [Authentication and approval](concepts/auth-and-approval.md) — when popups fire, what state the wallet needs to be in, how rejection surfaces.
