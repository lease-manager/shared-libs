# Lease transfer formats

Platform-neutral specification of the two ZIP archives exchanged between the desktop app and the
mobile apps. Implementations: desktop `app/ZipLeasePackageCodec` and `app/SignedReturnCodec`;
Android `transfer/` package.

Common rules: UTF-8 JSON, exactly the listed entries (no directories, no duplicates, no other names),
`manifest.json` <= 1 MiB, PDF <= 50 MiB. Hashes are lowercase hex SHA-256 of the raw file bytes.

## Trust model: pairing and Ed25519 signatures

Every device (desktop or phone) owns an Ed25519 key pair and a random device ID (UUID). Devices are
**paired once**: each shows its pairing code and the other side stores it. Packages and returns are
only accepted from paired devices.

Pairing code (shown as QR code, JSON, <= 2 KiB):

```json
{"v":"leasepair:v1","id":"<device UUID>","name":"<device name>","publicKey":"<Base64 X.509 SubjectPublicKeyInfo>"}
```

The public key is the standard X.509 encoding (`SubjectPublicKeyInfo`) of an Ed25519 key, Base64 (no line
breaks). Signatures are the raw 64-byte Ed25519 signature over the UTF-8 bytes of the *signed payload*,
Base64. Both users compare the short **fingerprint** of a key (first 8 bytes of SHA-256 of the decoded
public key, upper-case hex in groups of four, e.g. `1A2B 3C4D 5E6F 7081`) when pairing.

## `.leasepack` (desktop -> mobile), `schemaVersion = "leasepack:v2"`

| Entry | Content |
|---|---|
| `manifest.json` | `schemaVersion`, `contractId` (UUID), `createdAt` (ISO-8601 instant), `originalPdfSha256`, `contractName`, `contractJson` (string, desktop-internal contract data, JSON), `senderId` (device UUID of the desktop), `packageSignature` (Base64 Ed25519) |
| `contract.pdf` | Original, stamped PDF (QR code + metadata `LeaseContractId`, `LeaseContentHash`) |

Signed payload: `schemaVersion|contractId|createdAt|originalPdfSha256|contractName|contractJson|senderId`
(joined with `|`, `createdAt` exactly as in the manifest). A mobile app checks the signature with the public
key of the paired device whose ID equals `senderId`, and `SHA-256(contract.pdf) == originalPdfSha256`.

Legacy `leasepack:v1` (HMAC with a key local to one desktop, no `senderId`) is still read by the desktop on the
installation that created it; mobile apps cannot verify it and should reject it.

## `.leasesigned` (mobile -> desktop), `schemaVersion = "leasesigned:v2"`

| Entry | Content |
|---|---|
| `manifest.json` | `schemaVersion`, `contractId`, `originalPdfSha256` (copied from the received package), `signedPdfSha256`, `signedAt` (ISO-8601 instant, exactly as `Instant.toString()` would print it), `signerName` (string, may be empty), `deviceInfo` (string, may be empty), `signerId` (device UUID of the phone), `signature` (Base64 Ed25519) |
| `signed.pdf` | The contract with the drawn signature(s) |

Signed payload: `schemaVersion|contractId|originalPdfSha256|signedPdfSha256|signedAt|signerName|deviceInfo|signerId`
(joined with `|`; a missing `signerName`/`deviceInfo` counts as an empty string).

The desktop accepts a return only if `signerId` belongs to a paired device and the signature is valid,
`contractId` is known and `AWAITING_SIGNATURE`, `originalPdfSha256` equals the hash of the stored original, and
`SHA-256(signed.pdf) == signedPdfSha256`. Unsigned `leasesigned:v1` returns are rejected.
The signed PDF need not keep the original metadata (mobile apps may flatten pages to images);
the QR code stays visible on the page.

Example manifest: [`example-signed-manifest.json`](example-signed-manifest.json).
