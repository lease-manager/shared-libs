# Lease transfer formats

Platform-neutral specification of the two ZIP archives exchanged between the desktop app and the
mobile apps. Implementations: desktop `app/ZipLeasePackageCodec` and `app/SignedReturnCodec`;
Android `transfer/` package.

Common rules: UTF-8 JSON, exactly the listed entries (no directories, no duplicates, no other names),
`manifest.json` <= 1 MiB, PDF <= 50 MiB. Hashes are lowercase hex SHA-256 of the raw file bytes.

## `.leasepack` (desktop -> mobile), `schemaVersion = "leasepack:v1"`

| Entry | Content |
|---|---|
| `manifest.json` | `schemaVersion`, `contractId` (UUID), `createdAt` (ISO-8601 instant), `originalPdfSha256`, `contractName`, `contractJson` (string, desktop-internal contract data), `packageSignature` |
| `contract.pdf` | Original, stamped PDF (QR code + metadata `LeaseContractId`, `LeaseContentHash`) |

`packageSignature` is an HMAC-SHA-256 with a key local to one desktop installation. Mobile apps
**cannot verify it**; they only check `SHA-256(contract.pdf) == originalPdfSha256`
(integrity, not authenticity). A cross-device trust model (e.g. public-key signatures) is still open.

## `.leasesigned` (mobile -> desktop), `schemaVersion = "leasesigned:v1"`

| Entry | Content |
|---|---|
| `manifest.json` | `schemaVersion`, `contractId`, `originalPdfSha256` (copied from the received package), `signedPdfSha256`, `signedAt` (ISO-8601 instant), `signerName` (string, may be empty), `deviceInfo` (string, may be empty) |
| `signed.pdf` | The contract with the drawn signature(s) |

The desktop accepts a return only if `contractId` is known and `AWAITING_SIGNATURE`,
`originalPdfSha256` equals the hash of the stored original, and `SHA-256(signed.pdf) == signedPdfSha256`.
The signed PDF need not keep the original metadata (mobile apps may flatten pages to images);
the QR code stays visible on the page.

Example manifest: [`example-signed-manifest.json`](example-signed-manifest.json).
