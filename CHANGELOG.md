# Changelog

## Unreleased

### Security

- `resumableReceiver()` now enforces chunk and upload sizes, matching
  `singleShotReceiver()`. Previously a chunk sent without `Content-Length`
  (relying on `X-Upload-Content-Length`) was buffered without limit and its
  actual length was never checked, and the final `actualSize` was not compared
  with the prepared `size` — so an upload could exceed its declared size and
  `maxBytes` and still be marked `completed`.
  - Each chunk body is read up to its declared length. A longer body is cut off
    with `413`, a shorter one rejected with `400`; the record stays `pending` so
    the client can resend the chunk.
  - A `complete` outcome whose `actualSize` differs from the prepared `size` now
    fails the upload with `409 size mismatch` and runs destination `cleanup`.
    Custom `ResumableUploadDestination`s must report the number of bytes
    actually stored as `actualSize`.
- `createDownloads().serve()` adds `X-Content-Type-Options: nosniff` to download
  responses.

## 0.3.1

- `createDownloads().serve()` now falls back to the source response's
  `Content-Type` / `Content-Disposition` / `Content-Length` when the grant did
  not declare them. Grant values still win when present. This lets a backend
  that only learns the MIME type after fetching the bytes (e.g. by sniffing)
  advertise it without the grant having to know it up front.

## 0.3.0

### Breaking

The kit now covers downloads as well as uploads, so a few shared primitives were
renamed to transfer-neutral names:

- `createUploadId` → `createTransferId`
- `createUploadToken` → `createTransferToken`
- `uploadKey` → `transferKey`
- `UploadStore` → `TransferStore`
- `kvUploadStore` → `kvTransferStore`

Upload-specific API (`createUploads`, `UploadDestination`, `singleShotReceiver`,
`resumableReceiver`, `opaqueUploadToken`, `jwtUploadToken`, `signUploadJwt`, …)
is unchanged.

### Added

- `createDownloads({ store, baseUrl, ttlSeconds?, downloadPath? })` — the mirror
  image of `createUploads`. `prepare()` issues a short-lived signed download URL
  (not the bytes); `serve()` verifies the grant (bearer + expiry) and streams the
  bytes from a `DownloadSource` straight to the client, so large files never pass
  through the MCP channel or the model context.
- `DownloadSource`, `TransferDownloadRecord`, `DownloadGrant`,
  `DownloadPrepareInput`, `CreateDownloadsOptions`, `ServeDownloadInput` types.

## 0.2.0

- Make the `receiveWith` pipeline composable via a pluggable `UploadReceiver`.
  `createUploads({ receiver })` defaults to `singleShotReceiver()` (identical
  behaviour to before) and accepts `singleShotReceiver({ validate, verify })` to
  override only individual steps, or `resumableReceiver()` for chunked
  `Content-Range` uploads.
- Add `resumableReceiver()` plus the `ResumableUploadDestination` contract: the
  kit owns the `Content-Range`/308 wire protocol and record transitions, while
  the destination owns cross-request state (offset, accumulated bytes,
  end-to-end sha256) — typically a Durable Object. Adopting resumable uploads no
  longer requires reimplementing the whole handler.
- Tighten package build output to the library entrypoint.
- Document install, runtime assumptions, upload flow, token strategies, and
  security responsibilities.
- Add project security policy and license text.
