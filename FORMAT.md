# The Receipts record format

What a Receipts export is, written for someone who has never seen the
extension: a second implementer of the checker, or a second recorder that
wants to produce files this checker reads. Normative where it says MUST; the
test vectors under `vectors/` are the reference readings. Apache-2.0, like the
checker.

## 1. The file

An export is one HTML file. Everything the checker reads is inside one element:

```
<script type="application/json" id="fw-export-data">{ ...envelope... }</script>
```

The envelope is a JSON object with these keys, all present:

- `records`: the array of records, in the order the file presents them.
- `publicKeyJwk`: the file's own signing key, an EC P-256 public key as a JSON
  Web Key with at least `kty`, `crv`, `x`, `y`.
- `publicKeyFingerprint`: the first 16 hex characters of SHA-256 over the
  string `x|y` of that key (the two coordinates, base64url as in the JWK, with
  a vertical bar between). A checker MUST recompute it and refuse the file if it
  differs.
- `publicKeys`: every key the file carries, `[{ fingerprint, jwk }]`, the file's
  own first, then keys of earlier generations whose records are in the file.
  Each is checked for self-consistency the same way.
- `anchors`: independent time-stamp tokens over chain heads (section 6).
- `keyGenesis`: the time-stamp over the signing key itself, or null (section 6).
- `keyOrigins`, `signingKeyCreatedAt`, `selection`, `precisionDisclosure`,
  `fwVersion`, `locale`, `techLabels`, `i18n`: the file's own statements about
  itself and the wording its page renders. They are not evidence; they are
  covered by the manifest so they cannot be edited without notice.
- `manifest`: the signed manifest over the envelope (section 5), present in
  files made by builds from 2026-09-06 on. A file without it can read at most
  PARTIALLY VERIFIED.

A checker MAY accept the bare envelope as a JSON file. Without the HTML the
manifest's `provenanceDigest` cannot be checked (section 5), so such a file
reads at most PARTIALLY VERIFIED.

## 2. A record

Each record is a JSON object. The fields that carry evidence are the ones the
hash is taken over (section 3); everything else is display. Three fields sit
outside the hash:

- `integrityHash`: SHA-256, lowercase hex, over the canonical payload.
- `signature`: ECDSA P-256 with SHA-256 over the UTF-8 bytes of the
  `integrityHash` string (the hex text, not the digest bytes), base64url
  without padding, in the IEEE P1363 form Web Crypto produces (r then s, 32
  bytes each).
- `chainAnchor`: an advisory label; ignored by the checker.

Two fields inside the hash link the chain: `prevRecordHash`, the
`integrityHash` of the record before this one, and `chainPosition`, this
record's position, counting from 0 per generation. `publicKeyFingerprint`, also
inside the hash, names the key that signed the record.

`kind` names what the record is. Ordinary kinds describe content (`post`,
`comment`, `video`, `image`, `top-comment`, `environment`). Nine marker kinds
describe the chain's own housekeeping, each a signed record like any other,
whose `originalText` starts with `[<kind> @ <ISO time>]` followed by
space-separated `name=value` facts:

- `reset-marker`: the chain was reset; `priorChainHash` names the last record
  of the chain before.
- `month-rotation-marker`: the chain rolled into a new month's store;
  `priorChainHash` links the months.
- `prune-marker`: records were deleted under a size cap; `deleted=`, `cap=`.
- `gap-marker`: a record was lost between capture and the chain; `reason=`,
  `platform=` and the delivery id, so a hole is a stated hole.
- `heartbeat-marker`: the recorder was alive over a window; `from=`, `to=`, and
  the count of records skipped by the reader's own list without a name.
- `import-marker`: records from an earlier export were brought in;
  `imported=`, `alreadyPresent=`.
- `repair-marker`: stored positions were repaired; `ran=`, `method=`.
- `redaction-marker`: a record was withheld from exports at the person's
  request: `record=<id> recordHash=<hex> fields=<comma list>`; section 4.
- `reconcile-marker`: the record was compared with the platform's own export;
  `archive=<SHA-256 of that archive>`.

A record MAY carry `redacted: { markerId, at, fields }` and lack the fields
named; section 4.

A content record MAY carry `decision`, the record's own account of why the
product acted: `judge` (id, hash, model, provider, or null when no judge ran),
`reading` (what the judge returned, or null), `descriptor` (the description
call's fields, or null), `rails` (the deterministic rules that fired, each with
a short match token), `household` (mode, reader, the topic rule that applied or
null, whether swearing was masked), `row` (the policy row that fired), `action`
(what the policy said), `applied` (what the page showed), `rewrite` (attempted
and outcome, or null) and `note` (one sentence when `applied` differs from
`action`, else null). A record without `decision` states nothing about why; it
is not an allowance. On a record that carries `decision`, `verdict` is derived from
`applied`: allow, context, mask and record are `pass`; mark and soften are `flagged`;
collapse and hide are `flagged-hidden`. The checker does not judge any of it (section 7); it shows
`row` and `action` on the card.

A record MAY carry `reader`: the profile label declared at the keyboard when it
was written. A declared reader is a household's own label, not proof of who sat
there; the export says so beside every count grouped by it.

## 3. The canonical payload and the hash

`integrityHash` is SHA-256 over the UTF-8 bytes of `JSON.stringify` of an
object with exactly these keys in exactly this order, values taken from the
record as follows. A key whose value is `undefined` is omitted by
`JSON.stringify`; that omission is part of the format (it is how records
written before a field existed keep their hash). The four defaults marked
below are applied before serialisation.

```canonical-fields
id
timestamp
platform
originalText
originalHash
originalLength
neutralizedText
author
postUrl
techniques
overallScore
userAgeCategory
aiSource
feedSource
aiModel
aiProvider
detectionMode
configSnapshot
prevRecordHash
chainPosition
verdict
skipReason
kind
parentVideoId
environment
priorChainHash
resetReason
publicKeyFingerprint
topic
sourceCategory
parentRecordId
commentDepth
roomAuthor
roomCaption
roomUrl
harmFloor
failureKind
cap
metrics
anchorUrl
overrideOrigin
visualApplied
videoContext
accountType
sourceContainer
observedSequence
viewportSeenAt
surfaceSessionId
imageContext
surfaceOrigin
textCapture
mediaUrls
decision
reader
buildId
```

Defaults: `aiModel`, `aiProvider` and `detectionMode` default to `""`;
`configSnapshot` defaults to `null`; `prevRecordHash` defaults to `""`;
`chainPosition` defaults to `-1`. Every other key is passed through as it is,
`undefined` included. Nested values (`techniques`, `configSnapshot`,
`environment`, `metrics`, `mediaUrls` and the rest) are serialised as
`JSON.stringify` serialises them, with their own key order as stored. The
serialisation has no whitespace.

A new field MUST be appended at the end of this list, never inserted, and a
record written before the field existed MUST hash byte-identically after.

## 4. The checks, and the three readings

For each record, in file order:

1. Withheld record: if the record carries `redacted`, its content is gone and
   its hash cannot be recomputed. It is accepted only when a `redaction-marker`
   in the same file names its `id` and its `integrityHash` and that marker's
   own signature verifies; otherwise it fails with `redaction-unattested`.
2. Hash: recompute the canonical payload and compare with `integrityHash`. A
   mismatch fails with `tampered`.
3. Signature: without a `signature` the record is `unsigned` (accepted, never
   green). Otherwise look up the key by `publicKeyFingerprint` (the file's own
   key when absent). A key not in the file gives `prior-key` (accepted by hash
   only). A signature that does not verify under the named key fails with
   `key-mismatch`. A signature that verifies under a key that is not the file's
   own is `carriedKey` (accepted, never green: the file cannot say whose key
   it is).

Then the chain: every `prevRecordHash` is resolved against every record in the
file. A predecessor present is a link. A predecessor absent where positions are
consecutive is a seam (a restart; counted, attributed by signature where both
sides verify). A position held twice is a branch (counted). A predecessor absent
where positions are not consecutive is a record outside the file (counted).
None of these fails a file on its own; a modified record fails its own hash and
a forged one its own signature.

Then the anchors: a time-stamped record hash absent from the file, or a
time-stamped position past the last record, is counted (section 6).

The reading:

- **failed** (BROKEN): any record failed, or the manifest failed.
- **partial** (PARTIALLY VERIFIED): nothing failed but something is `unsigned`,
  `prior-key` or `carriedKey`; or there is a seam, a branch or a record outside
  the file that the file's own selection does not account for; or an anchor
  names something not here; or there is no manifest.
- **verified** (VERIFIED): none of the above, and the manifest verifies. A
  report reads VERIFIED as a report when its `selection` block (under the
  manifest) says `included` equals the records present and the records
  outside the file are no more than `totalSource - included`: those missing
  links are the report's own doing, stated under the signer's key, not a loss.
  A checker MUST still print the count and that the file is a selection.

A checker SHOULD say first what the reader is holding: a report of `included`
of `totalSource` records, or the whole record; when it was made
(`manifest.generatedAt`), by which build (`fwVersion`), under which key
(`publicKeyFingerprint`), and whether the file carries any independent time
stamp at all (`anchors`, `keyGenesis`). A file with none is not wrong, and
must not read as if the stamps were checked.

A withheld record on a good marker is `verified`, not partial: the marker is
the person's signed statement and the reader is told a field is withheld.

## 5. The signed manifest

`manifest` is `{ version: 2, recordCount, recordsDigest, anchorsDigest,
publicKeysDigest, techLabelsDigest, i18nDigest, publicKeyFingerprint,
generatedAt, envelopeDigest, provenanceDigest, digest, signature }`. Every
digest is SHA-256, lowercase hex:

- `recordsDigest` over `JSON.stringify(records)`; `anchorsDigest`,
  `publicKeysDigest`, `techLabelsDigest`, `i18nDigest` likewise over those
  envelope values (`[]` or `{}` when absent).
- `envelopeDigest` over `JSON.stringify([selection, keyGenesis, keyOrigins,
  signingKeyCreatedAt, precisionDisclosure, fwVersion, locale, publicKeyJwk])`
  with `null` for any absent value.
- `provenanceDigest` over `JSON.stringify` of the page's provenance rows: every
  `<dd data-fw-prov="name">` AND every `<p data-fw-prov="name">` element in the
  HTML, as `[name, text]` pairs sorted by name, `text` being the element's text
  content with tags dropped, entities decoded, whitespace collapsed and trimmed.
  This is why a bare JSON envelope cannot fully verify. A checker reading one
  MUST report that comparison as not made and cap its verdict at partially
  verified — it must not report it as failed. A bare envelope has no page rows,
  so a digest over zero rows can never equal the manifest's digest over the
  page's rows, and treating that as a mismatch tells a recipient a sound file is
  broken. Everything else in the manifest still applies: `recordsDigest` and
  `envelopeDigest` are checked before this step, so a record removed from or
  reordered in a bare envelope still fails.

  Both tags, and only those two. The summary rows are `<dd>` inside the
  definition list; the disclosure block — the paragraph naming what the file
  reveals about the person who made it — is a `<p>`. A checker matching `<dd>`
  alone leaves that one paragraph outside the signature, which is the one
  summary claim an editor could then rewrite with nothing breaking. A checker
  matching *any* tag would disagree with the builder the first time a third
  element carries the attribute, so the pair is named explicitly here and in all
  three shipped implementations.

  Entities decoded, in this order: `&#39;` `&quot;` `&lt;` `&gt;` `&amp;`.
  `&amp;` must be decoded LAST, or `&amp;lt;` collapses to `<` instead of
  `&lt;`.
- `digest` over `JSON.stringify([version, recordCount, recordsDigest,
  anchorsDigest, publicKeysDigest, techLabelsDigest, i18nDigest,
  publicKeyFingerprint, generatedAt, envelopeDigest, provenanceDigest])`.
- `signature` as for a record, over the UTF-8 bytes of `digest`, under the
  file's own key.

A checker MUST check the manifest before printing any verdict wording, since
the wording itself (`i18n`) is under it. A manifest with `version: 1` (files of
one morning in September 2026) reads as absent.

## 6. Time stamps

`anchors[]` are RFC 3161 time-stamp responses obtained from public authorities
over a chain head: `{ anchoredChainPosition, anchoredIntegrityHash,
tsaProvider, responseB64, tstB64, generalizedTime, requestedAt }`. The token is
base64 of the full TimeStampResp (or of the bare TimeStampToken in `tstB64`).
A checker opens it offline: the imprint MUST be SHA-256 of
`anchoredIntegrityHash`'s UTF-8 text; the signature and the certificate chain
inside the token are checked; the root certificate's SHA-256 is compared with
the authorities the recorder pins. A token that verifies under an unrecognised
root is reported as such and MUST NOT date anything on its own: anyone can run
a timestamping service.

`keyGenesis` is the same kind of token over the signing key: `{ publicKeyDigest,
fingerprint, anchors: [...] }` where the imprint MUST be SHA-256 of the string
`x|y` of `publicKeyJwk`, recomputed from the key in the file, never read from
`publicKeyDigest`. The earliest verified time under a recognised authority is
the time the key is known to have existed by.

The `generalizedTime` and `requestedAt` fields are what the recorder stored;
the time a checker reports is the one read out of the signed token.

## 7. What the format does not claim

A file that reads VERIFIED is a file whose records have not changed since the
key in it signed them, and whose envelope is the one that key signed. The
format does not establish who holds the key, that the key is the one the
recorder generated, that the file holds every record the chain held, that a
post existed on the platform, or that any judgement a record carries is right.
Each of those has its own evidence, some of it in the file (the time stamps,
the reconcile marker, the stated selection), none of it proof.
