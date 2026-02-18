---
name: atproto-keygen
description: Generate a secp256k1 keypair suitable for AT Protocol (ATproto). Outputs privateKeyHex, publicKeyMultibase, and a did:key URI. Invoke when the user wants to create ATproto signing keys, a PLC operation key, or a did:key for Bluesky/ATproto use.
user-invocable: true
argument-hint: [label-or-purpose]
allowed-tools: Bash
---

# ATproto Keypair Generator

Generates a **secp256k1** keypair compatible with the AT Protocol spec:
- Private key: raw 32-byte scalar, hex-encoded
- Public key: compressed point with `0xe701` multicodec prefix, base58btc multibase (`z…`)
- DID: `did:key:z…` URI ready for use in PLC operations or repo signing

## How to invoke

Run the following Node.js snippet (no external dependencies):

```js
const { generateKeyPairSync } = require('crypto');

const BASE58 = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';
function b58(buf) {
  let n = BigInt('0x' + buf.toString('hex')), s = '';
  while (n > 0n) { s = BASE58[Number(n % 58n)] + s; n /= 58n; }
  for (const b of buf) { if (b !== 0) break; s = '1' + s; }
  return s;
}

const PREFIX = Buffer.from([0xe7, 0x01]);
const { privateKey, publicKey } = generateKeyPairSync('ec', { namedCurve: 'secp256k1' });
const jwk = privateKey.export({ format: 'jwk' });
const der = publicKey.export({ type: 'spki', format: 'der' });
const pt = der.slice(-65);
const compressed = Buffer.concat([Buffer.from([(pt[64] & 1) ? 0x03 : 0x02]), pt.slice(1, 33)]);
const multibase = 'z' + b58(Buffer.concat([PREFIX, compressed]));
console.log(JSON.stringify({ curve:'secp256k1', privateKeyHex: Buffer.from(jwk.d,'base64url').toString('hex'), publicKeyMultibase: multibase, didKey: 'did:key:' + multibase }, null, 2));
```

When invoked, execute this snippet via `node -e '...'` using the Bash tool and display the result to the user.

Always remind the user to store the `privateKeyHex` securely (env var or secret manager) and never commit it to version control.

If $ARGUMENTS is provided, use it as a label or note to describe the key's purpose when presenting the output.
