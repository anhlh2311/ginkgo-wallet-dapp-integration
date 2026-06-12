# Accounts

Methods for discovering the wallet user's identity. Both are read-only and do not trigger approval popups.

## `listAccounts`

Returns all accounts the user has configured. Currently always at most one (Ginkgo is single-account; see [extensions](../extensions/ginkgo-vs-cip-0103.md#single-account)).

### Params

None.

### Returns

```ts
DappAccount[]
```

Empty array when the wallet is locked or has no party onboarded.

### Errors

None.

### Example

```ts
const accounts = await provider.request({ method: 'listAccounts', params: {} });
if (accounts.length === 0) {
  alert('Please unlock the Ginkgo wallet.');
}
```

## `getPrimaryAccount`

Returns the single primary account. Throws if no account is available.

### Params

None.

### Returns

```ts
DappAccount = {
  primary: boolean;                                  // always true here
  partyId: string;                                   // 'hint::fingerprint'
  status: 'initialized' | 'allocated' | 'removed';   // always 'allocated' in current Ginkgo
  hint: string;                                      // pre-'::' portion
  publicKey: string;                                 // base64-encoded Ed25519 public key
  namespace: string;                                 // post-'::' portion (the fingerprint)
  networkId: string;                                 // matches getActiveNetwork().networkId
  signingProviderId: string;                         // 'ginkgo'
  disabled?: boolean;
  reason?: string;
}
```

### Errors

- `4100 UNAUTHORIZED` — wallet is locked or no party is onboarded.

### Example

```ts
const account = await provider.request({ method: 'getPrimaryAccount', params: {} });
console.log(`Signed in as ${account.partyId}`);
console.log(`Public key: ${account.publicKey}`);
```

### When to call

Call `getPrimaryAccount` **once on connect** to capture the `publicKey` for local signature verification. The `publicKey` does not change between calls by the same wallet, so cache it for the session. See [Verify a signature](../guides/verify-signatures.md) for the canonical pattern.

```ts
// At app startup or after connect:
const account = await provider.request({ method: 'getPrimaryAccount', params: {} });
const cachedPublicKey = account.publicKey;

// Subsequent signMessage calls return only { signature }; reuse cachedPublicKey to verify.
```

### Notes

- `partyId` is the canonical Canton identifier. Format is `<hint>::<fingerprint>`, where `hint` is human-chosen (often the user's email local-part or wallet handle) and `fingerprint` is a deterministic hash of the public key registered with the Canton synchronizer.
- `publicKey` is base64-encoded as standard base64 (with `=` padding). 32 raw bytes → 44 characters base64.
- `signingProviderId` is `'ginkgo'`. The SDK uses this to route signing-relay traffic; for direct dApp usage, you can ignore it.
- `namespace === partyId.split('::')[1]` (always; pre-split for convenience).
- `hint === partyId.split('::')[0]` (always).
