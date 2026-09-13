# verify-export: a checker for FeelingWise Receipts exports

Receipts is a Chrome extension that keeps, on the user's own computer, a signed
and hash-linked record of the public posts a social feed showed them. It
exports that record as one HTML file that carries its own checker. This folder
holds the same checker as a separate program, so a reader can obtain it from a
place they chose instead of trusting the script inside the file they were
handed.

## Run it

Two ways, the same checks.

**In a browser.** Open `index.html` from this folder (double-click it; it works
from a local file with the network off) and choose or drop the export file. The
page loads `verify-export.browser.js` beside it and nothing else, and makes no
network request.

**From the command line.** Needs Node 20 or later. No installation, no
dependencies, no network.

```
node verify-export.mjs path/to/feelingwise-export.html
node verify-export.mjs path/to/feelingwise-export.html --json
```

Exit codes: 0 verified, 1 partially verified, 2 broken, 3 the file could not
be read as an export.

## What it checks

For every record: the integrity hash is recomputed from the record's canonical
fields and compared with the stored one; the signature over that hash is
checked under the public key whose fingerprint the record names, among the
keys the file carries. Then the chain links between records are walked, records
withheld from the file are checked against their signed redaction markers, the
independent timestamps are compared with the records present, and the signed
manifest over the whole envelope (records, keys, labels, wording, the rows on
page one as rendered) is recomputed and its signature checked under the file's
own key. Then every RFC 3161 time-stamp token the file carries is opened
offline, the chain anchors against the record hash each names and the stamp
over the signing key against a digest recomputed from the key in the file: the
imprint, the signature, the certificate chain inside the token, and whether
the root is one of the authorities the extension pins. A token that verifies
under an authority the checker does not recognise is reported as such and
dates nothing on its own, because anyone can run a timestamping service.

The three readings:

- **VERIFIED**: every hash matched, every signature verified, the manifest
  matches and is signed by the file's own key.
- **PARTIALLY VERIFIED**: nothing failed, but something could not be checked.
  The reasons are printed: records whose signing key is not in the file, records
  with no signature, a file made before the manifest existed, a key that
  travelled inside the file, a chain seam, a timestamped record that is missing.
- **BROKEN**: a hash or a signature did not match, or the manifest did not.
  The first failing record and the reason are printed.

## What it cannot establish

The checker compares the file with itself. It does not establish who created
the file, whether the embedded key is the original key, that the file holds
everything the record held, or that anything a record says is true. A file that
reads VERIFIED is a file whose contents have not changed since the key that
signed them signed them. The extension's own methodology, and the measured
accuracy of the optional AI reading, are described inside every export and at
feelingwise.org/research.

## Receiving donated exports

Receipts carries a "Send this export to <organisation>" button that POSTs the
export, from the person's own computer, to one HTTPS address, with no account
and no identifier of the sender beyond the public key already inside the file.
`receive-export.mjs` is a reference for the other end, for an organisation that
has agreed to receive exports and is the data controller for what it keeps:

```
node receive-export.mjs --dir ./received --port 8788
```

It accepts one POST per file, runs the checker above over it, keeps the file
and its reading under a name that is the file's own digest (the same file sent
twice is kept once), and answers `200` with an `X-Receipt` header. Not an
export: `422`. Over the size limit (`--max-mb`, 64 by default): `413`. It logs
no address, header or cookie. Put it behind a reverse proxy that terminates
HTTPS; the extension refuses to send to an `http` address.

## Test vectors, for a second implementer

`vectors/` holds nine small exports and `EXPECTED.json`, the reading this
checker gives each: valid with its manifest; without the manifest; a record
edited; the manifest's file edited; a withheld record on its signed marker; an
earlier generation's key carried; that key missing; a self-minted time-stamp
token; the signing key dated by FreeTSA. Every record in them is invented and
the file says so. A program that implements this checker, or a recorder that
produces this format, can run over these files and compare its readings to
`EXPECTED.json`. They are built once by `scripts/build-verifier-vectors.ts` in
the FeelingWise tree and rebuilt only when what a file carries changes.

## How it is built

`verify-export.mjs` is generated, not written by hand. The functions between
the CORE markers are the exact text every export embeds in its own verifier,
taken from `src/forensics/verifier-core-source.ts` in the FeelingWise tree by
`scripts/build-standalone-verifier.ts`; the RFC 3161 block is the same compiled
copy of `src/forensics/rfc3161.ts` the export embeds. A test in that tree fails whenever the
committed file differs from the builder's output, and runs this file over a
real signed export, a copy without its manifest, and two tampered copies.

The record format, the canonical field order the hash is taken over, the
chain, the marker kinds, the manifest and the time stamps are specified for
a second implementer in `FORMAT.md` beside this file, normative against the
vectors; the tree's own working spec is `docs/spec/subsystems/forensic-chain.md`.

## Licence

Apache License, Version 2.0. See `LICENSE`.
