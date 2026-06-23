# Signing

Local Ed25519 signing methods. No network round-trip — the wallet decrypts the private key in memory and signs directly. Both methods trigger an approval popup.

## `signMessage`

Sign an arbitrary UTF-8 string with the user's Canton private key. Spec-canonical: `nacl.sign.detached(utf8(message), privateKey)`.

### Params

```ts
{ message: string }
```

### Returns

```ts
{ signature: string }  // base64-encoded 64-byte Ed25519 signature
```

CIP-0103 mandates this exact shape — `{ signature }` and only `{ signature }`. To verify the signature locally, the dApp sources the `publicKey` separately from `getPrimaryAccount` and caches it. See [Verify a signature](../guides/verify-signatures.md).

### Errors

- `4001 USER_REJECTED` — user declined the approval popup.
- `4100 UNAUTHORIZED` — wallet is locked or no party is onboarded.
- `-32602 INVALID_PARAMS` — `message` missing, not a string, or empty.

### Example

```ts
import { listAccounts, signMessage } from '@canton-network/dapp-sdk';
import nacl from 'tweetnacl';
import naclUtil from 'tweetnacl-util';

// Cache the publicKey once.
const [primary] = await listAccounts();

// Sign.
const message = 'Sign me to prove key ownership.';
const { signature } = await signMessage({ message });

// Verify with any standard Ed25519 implementation.
const verified = nacl.sign.detached.verify(
  new TextEncoder().encode(message),
  naclUtil.decodeBase64(signature),
  naclUtil.decodeBase64(primary.publicKey),
);
```

### Notes

- The pre-image is the **UTF-8 bytes of `message`** — *not* a hash, *not* prefixed with anything. Compatible with `tweetnacl`, `libsodium`, Web Crypto's `Ed25519` algorithm, and Canton's own kernel verifier.
- The signature is 64 bytes (raw), base64-encoded → 88 characters with trailing `==` padding.
- Approval popup shows the full message text. If your dApp signs sensitive payloads, keep them human-readable; users have to read what they're signing.

## `signTransaction`

Sign a 32-byte transaction hash. **Ginkgo extension** — not part of CIP-0103. Provided for use cases where the dApp computes its own prepared-transaction hash and wants only the signature, without going through the backend-mediated `prepareExecute` flow.

If you don't have an existing reason to use this, prefer `prepareExecute` instead — it handles the prepare/sign/submit lifecycle correctly and matches the spec.

### Params

```ts
{ transactionHash: string }  // base64-encoded 32-byte hash
```

### Returns

```ts
{
  signature: string;    // base64-encoded 64-byte Ed25519 signature
  publicKey: string;    // base64-encoded Ed25519 public key (echo of the active account)
  fingerprint: string;  // the partyId's fingerprint portion (post-'::')
}
```

This shape is a Ginkgo convenience extension (the spec has no equivalent method). The extra fields save the dApp a `getPrimaryAccount` round-trip when verifying.

### Errors

- `4001 USER_REJECTED` — user declined the approval popup.
- `4100 UNAUTHORIZED` — wallet is locked or no party is onboarded.
- `-32602 INVALID_PARAMS` — `transactionHash` missing, not a string, or fails the base64 validation.

### Input validation

`transactionHash` must be a valid base64 string. To prevent a class of silent bugs where a hex-encoded hash gets base64-decoded into garbage and signed, Ginkgo also rejects strings that look hex-only:

| Input | Result |
|---|---|
| `'WeYM7eKGm3jVPx4Tatbb9lzXUBiEoz6aRgshDht8Pgs='` (32 bytes b64) | ✅ accepted |
| `'aabbccdd…' * 8` (64-char lowercase hex) | ❌ rejected: `transactionHash must be base64-encoded` |
| `'not!base64?'` | ❌ rejected: `transactionHash must be base64-encoded` |
| `''` | ❌ rejected: `Missing or invalid transactionHash parameter` |

### Example

```ts
import { getConnectedProvider } from '@canton-network/dapp-sdk';
import nacl from 'tweetnacl';
import naclUtil from 'tweetnacl-util';

const hashBytes = new Uint8Array(32);
crypto.getRandomValues(hashBytes);
const transactionHash = naclUtil.encodeBase64(hashBytes);

// signTransaction isn't a CIP-0103 method — it's a Ginkgo extension and not
// exposed as a top-level SDK helper. Reach for the underlying provider.
const provider = getConnectedProvider();
if (!provider) throw new Error('Not connected');
const { signature, publicKey } = await provider.request({
  method: 'signTransaction',
  params: { transactionHash },
});

const verified = nacl.sign.detached.verify(
  hashBytes,                              // raw 32 bytes — the pre-image is the bytes, NOT utf8(transactionHash)
  naclUtil.decodeBase64(signature),
  naclUtil.decodeBase64(publicKey),
);
```

### Pre-image semantics

The signature is over the **raw bytes** that `transactionHash` decodes to — not the UTF-8 encoding of the base64 string. This matches what Canton expects when verifying a signature over a `preparedTransactionHash`.

In code: `nacl.sign.detached(base64Decode(transactionHash), privateKey)`.

### Notes

- For Canton transaction signing in practice, use `prepareExecute` instead. It calls `signTransaction`-equivalent logic internally with a real prepared hash from the backend, and handles submission. `signTransaction` is a primitive for advanced cases.
- The approval popup displays the base64 hash string. Users should treat any prompt to sign a raw hash with caution — it's intentionally less informative than a `prepareExecute` prompt and harder to audit.
