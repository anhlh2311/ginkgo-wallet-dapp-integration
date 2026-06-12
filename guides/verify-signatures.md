# Verify a signature

How to verify a signature returned by `signMessage` (or `signTransaction`) against the wallet user's `publicKey`. Three common stacks — pick the one your dApp already uses.

## What you're verifying

After `signMessage({ message })`:

- The signature is a base64-encoded 64-byte Ed25519 signature.
- The **pre-image** is the UTF-8 byte representation of `message` directly — no hashing, no prefix.
- The **public key** is base64-encoded 32 bytes, sourced from `getPrimaryAccount().publicKey`.

Verification is purely local — no network round-trip, no wallet interaction. Any standards-compliant Ed25519 verifier will accept the signature.

## Pattern: cache the publicKey once

`signMessage` returns `{ signature }` only — no `publicKey` in the response (per CIP-0103). Fetch it once on connect and reuse it across all subsequent signatures.

```ts
let cachedPublicKey: string | null = null;

async function connectAndCache() {
  await provider.request({ method: 'connect', params: {} });
  const account = await provider.request({ method: 'getPrimaryAccount', params: {} });
  cachedPublicKey = account.publicKey;
}

// On disconnect or accountsChanged event, clear the cache:
function onDisconnect() {
  cachedPublicKey = null;
}
```

The publicKey doesn't change between signs by the same wallet. If the user switches networks or accounts, refresh.

## Stack A — `tweetnacl` (smallest, browser+Node)

```bash
npm install tweetnacl tweetnacl-util
```

```ts
import nacl from 'tweetnacl';
import naclUtil from 'tweetnacl-util';

function verifySignMessage(
  message: string,
  signature: string,
  publicKey: string,
): boolean {
  return nacl.sign.detached.verify(
    new TextEncoder().encode(message),
    naclUtil.decodeBase64(signature),
    naclUtil.decodeBase64(publicKey),
  );
}

// Usage:
const { signature } = await provider.request({
  method: 'signMessage',
  params: { message: 'Hello' },
});
const ok = verifySignMessage('Hello', signature, cachedPublicKey!);
```

About 50KB unminified, ~6KB gzipped. Works identically in browser and Node 20+. No transitive deps.

## Stack B — `libsodium-wrappers` (largest, fastest)

```bash
npm install libsodium-wrappers
```

```ts
import sodium from 'libsodium-wrappers';
await sodium.ready;

function verifySignMessage(
  message: string,
  signature: string,
  publicKey: string,
): boolean {
  return sodium.crypto_sign_verify_detached(
    sodium.from_base64(signature, sodium.base64_variants.ORIGINAL),
    message,                                // sodium accepts strings directly; encodes as utf8
    sodium.from_base64(publicKey, sodium.base64_variants.ORIGINAL),
  );
}
```

Larger (~280KB) but battle-tested, native-equivalent performance via WASM. Best for high-volume verification (e.g., a server validating many signatures).

## Stack C — Web Crypto (zero dependencies, browser+Node 20+)

```ts
function decodeBase64(b64: string): Uint8Array {
  const bin = atob(b64);
  const bytes = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
  return bytes;
}

async function verifySignMessage(
  message: string,
  signature: string,
  publicKey: string,
): Promise<boolean> {
  const key = await crypto.subtle.importKey(
    'raw',
    decodeBase64(publicKey),
    { name: 'Ed25519' },
    false,
    ['verify'],
  );
  return crypto.subtle.verify(
    'Ed25519',
    key,
    decodeBase64(signature),
    new TextEncoder().encode(message),
  );
}
```

Zero install. Requires browser support for Ed25519 in Web Crypto, which landed in Chrome 113+, Firefox 130+, Safari 17+, and Node 20+ (behind `--experimental-vm-modules` in some versions). For older targets, fall back to `tweetnacl`.

## Verifying `signTransaction` output

`signTransaction({ transactionHash })` signs the **raw 32 bytes** that `transactionHash` decodes to — not the UTF-8 of the base64 string. The verifier looks slightly different:

```ts
import nacl from 'tweetnacl';
import naclUtil from 'tweetnacl-util';

function verifySignTransaction(
  transactionHash: string,  // base64-encoded 32 bytes
  signature: string,
  publicKey: string,
): boolean {
  return nacl.sign.detached.verify(
    naclUtil.decodeBase64(transactionHash),   // decode FIRST — sig is over the raw bytes
    naclUtil.decodeBase64(signature),
    naclUtil.decodeBase64(publicKey),
  );
}
```

Note the difference vs `signMessage`: pre-image is `decodeBase64(transactionHash)`, not `new TextEncoder().encode(transactionHash)`.

## End-to-end test

```ts
// Full self-contained smoke test. Paste into a dApp page's DevTools console
// after connecting to Ginkgo.

import nacl from 'tweetnacl';
import naclUtil from 'tweetnacl-util';

const message = 'verifying-' + Math.random();

const account = await provider.request({ method: 'getPrimaryAccount', params: {} });
const { signature } = await provider.request({
  method: 'signMessage',
  params: { message },
});

const verified = nacl.sign.detached.verify(
  new TextEncoder().encode(message),
  naclUtil.decodeBase64(signature),
  naclUtil.decodeBase64(account.publicKey),
);

console.assert(verified, 'signature should verify');
console.log(`✅ verified: ${verified}`);
```

## Common pitfalls

- **Passing the base64 string to `nacl.sign.detached.verify` directly.** The function expects `Uint8Array`s. Decode first.
- **URL-safe base64.** Ginkgo emits standard base64 (with `+`, `/`, `=`). If your stack expects URL-safe (`-`, `_`, no padding), transform first: `signature.replace(/-/g, '+').replace(/_/g, '/')` and pad to a multiple of 4 with `=`.
- **Hashing the message before verification.** `signMessage` signs the raw UTF-8 bytes. Don't sha256 the message before passing it to the verifier.
- **Using the wrong key.** If the user has multiple wallets installed (multi-extension), make sure you cached the publicKey from the *Ginkgo* `getPrimaryAccount` call, not another wallet's.
