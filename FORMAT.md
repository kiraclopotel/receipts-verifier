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
- `profile`: who made the file and on what machine, `{ name, platform, browser,
  language, timeZone }`, any of them null, or the whole block null. Present in
  files made by builds from 2026-09-14 on. It is the one envelope key NO digest
  covers: `envelopeDigest` was fixed before this key existed and adding an
  element to its canonical array would refuse every file signed under the old
  order. It is held down the other way instead. The page renders these same
  values as the provenance rows `madeBy` and `machine`, which are under
  `provenanceDigest`, and a checker MUST refuse a file whose `profile`
  disagrees with them: a non-empty `name` requires a `madeBy` row beginning with
  that name, an absent or empty `name` requires no `madeBy` row, a `profile`
  present at all requires a `machine` row, and when at least one of the four
  machine facts is stated the `machine` row MUST equal those facts joined with
  `, ` in the order platform, browser, language, time zone. With all four absent
  the page prints a sentence in the file's own language and there is nothing to
  compare. A checker that cannot see the page rows at all (a bare envelope) does
  not make this comparison, for the reason given under `provenanceDigest`.
  `profile` is never evidence of who made anything: section 7 applies to it as
  it applies to the key.
- `unreadableArchivedRows`: how many archived records the install holds and
  could not read when it made the file, so they are not in it. Present only in
  a whole-record file that has such records, and absent otherwise, so every
  other file is exactly as it was before the key existed. Like `profile` it is
  under no digest, for the same reason, and it is held down the same way: the
  page renders it as the provenance row `unreadableRows`, whose text begins with
  the count, under `provenanceDigest`. A checker MUST take the count only when
  the manifest verified and that row begins with the same whole number, and
  otherwise read the file as if the key were absent. The count names a cause
  and never changes a reading: section 4 says what it explains.
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
describe the chain's own housekeeping, each a signed record like any other.
Eight of them have an `originalText` that starts with `[<kind> @ <ISO time>]`
followed by space-separated `name=value` facts, always in the order given here.
The ninth, the month rotation, is shaped differently and is given in full. Any
of the nine may end with the bare word `settings-read-failed`: the marker was
written while the settings could not be read, and it says so rather than attest
a setting nobody chose. A reader takes each fact by name and ignores a name it
does not know, because facts are only ever appended.

- `reset-marker`: the chain was reset. The text carries no facts:
  `[reset-marker @ <ISO time>]`. What it links to is a field of the record, not
  part of the text: `priorChainHash` names the last record of the chain before.
- `month-rotation-marker`: the chain rolled into a new month's store. Its text
  does not use the kind as its tag and keeps its one fact inside the brackets:
  `[month-rotation @ <ISO time> from=<YYYY-MM>]`, where `from` is the month that
  was archived. `priorChainHash`, a field of the record and not part of the
  text, links the months.
- `prune-marker`: records were deleted under a size cap: `deleted=`, `cap=`,
  then `positions=<first>..<last>` when the range is known, then `archived=`
  with the months the deleted records went to, or `archived=none`.
- `gap-marker`: something was lost between capture and the chain, so a hole is
  a stated hole: `reason=`, `platform=`, `count=`, `unit=`, `from=`, `to=`,
  and for a lost chain a breakdown of what went. The delivery id of the row
  that was given up on is NOT in the text; this document said it was until
  2026-09-20.
- `heartbeat-marker`: the recorder was alive over a window: `from=`, `to=`,
  `feedsOpen=`, `positionsSince=` (a number, or `chain-restarted`),
  `canary=`, and `excludedSkipped=`, the count of records skipped by the
  reader's own list. It is a count only: no account is named.
- `import-marker`: records from an earlier export were brought in:
  `imported=`, `alreadyPresent=`, `rejected=`, `keys=` (the signing keys of
  what came in, or `none`), `idsDigest=`.
- `repair-marker`: stored positions were repaired: `ran=`, `method=`,
  `repaired=`, `unrepaired=`, `healed=yes|no`, and `error=` when there was one.
- `redaction-marker`: a record was withheld from exports at the person's
  request: `record=<id> recordHash=<hex> fields=<comma list>`; section 4. This
  is the only marker whose text the checker parses.
- `reconcile-marker`: the record was compared with the platform's own export:
  `archive=<SHA-256 of that archive>`, `platforms=`, `from=`, `to=`,
  `windowMinutes=`, `corroboratedByUrl=`, `corroboratedByAuthorAndTime=`,
  `unmatchedRecords=`, `unmatchedArchiveItems=`, `recordsOutsideWindow=`, and,
  on markers written since 2026-09-18 and only when present,
  `systemMarkersExcluded=`, `archiveItemsOutsideCoverage=`, `comparedFrom=`,
  `comparedTo=`.

A record MAY carry `redacted: { markerId, at, fields }` and lack the fields
named; section 4.

A content record MAY carry `decision`, the record's own account of why the
product acted: `judge` (id, hash, model, provider, or null when no judge ran),
`reading` (what the judge returned, or null), `descriptor` (the description
call's fields, or null), `rails` (the deterministic rules that fired, each with
a short match token), `household` (mode, reader, the topic rule that applied or
null, whether swearing was masked), `row` (the policy row that fired), `action`
(what the policy said), `applied` (what the page showed), `rewrite` (attempted
and outcome, and `model`, the model whose rewrite the page showed, when a model
wrote it and its reply named the model; or null) and `note` (one sentence when `applied` differs from
`action`, else null). A record without `decision` states nothing about why; it
is not an allowance. On a record that carries `decision`, `verdict` is derived from
`applied`: allow, context, mask and record are `pass`; mark and soften are `flagged`;
collapse and hide are `flagged-hidden`. The checker does not judge any of it (section 7); it shows
`row` and `action` on the card.

A content record MAY carry `configSnapshot`, the settings in force when the
verdict was reached: `mode`, `threshold`, `dailyCap`, and optionally
`enabledFeatures` (the switches that were on), `settingsReadFailed`, and on
records written before 2.5.0 `videoUnderstanding` and `imageUnderstanding`. It is
inside the hash. Two of its values need saying, because a reader cannot tell
them from the numbers alone:

- `threshold` holds one of two things, by the path that wrote the record. A
  post record holds the mode's confidence cut, a value below 1 (0.45, 0.55 or
  0.60). A comment record holds a severity floor on the 1 to 10 scale (6). A
  value below 1 is a confidence cut; 1 or above is a severity floor. Printed
  as one scale they read as a tenfold difference that does not exist.
- The confidence on a record written by the adult reader is 0.9 on every
  record it acts on. It is a fixed value and not a measure of certainty: the
  rule that reader applies has no confidence term. The export says so in its
  methodology block. A checker reports it as written and draws nothing from it.

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
verdictOrigin
authorCapture
authorName
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

### The serialisation, in bytes

Naming `JSON.stringify` names a function in one language, and that is not a
specification. Measured on 2026-09-18 against `vectors/11-text-escaping.html`,
which was built for this: Python's standard `json.dumps` cannot produce the
digest of that file under ANY combination of its options, and an implementer who
reaches for it gets a false BROKEN on a sound file, which section 3 above already
calls the loudest possible wrong answer. So the rules are written out here, and
that vector is the one to test against.

- **UTF-8**, no byte order mark. No whitespace anywhere: no spaces after `:` or
  `,`, no newlines.
- **Escaping.** `"` and `\` are escaped. The C0 controls use the short forms
  `\b \f \n \r \t` where those exist and `\u00XX` otherwise, lowercase hex.
  `/` is NOT escaped. Every other character, including every non-ASCII character
  and every astral character, is emitted RAW as UTF-8, not as `\uXXXX`.
- **Lone surrogates**, which a social media post really can carry, are escaped as
  `\udXXX` (ECMAScript 2019 well-formed JSON.stringify). They are NOT emitted
  raw, and they are NOT replaced with U+FFFD. This is the rule that defeats
  Python's library outright: it needs `ensure_ascii=False` for the emoji and
  `\uXXXX` for the lone surrogate in the same pass, and no flag gives both.
- **Numbers** are printed by the ECMA-262 `Number::toString` algorithm, which is
  the rule and not "whatever the language prints". It is not obvious: `1e-7`
  prints `1e-7` and not `1e-07`, `-0` prints `0`, `1e21` prints `1e+21`. The
  exponent has no leading zero. Python's `repr` disagrees on the first of those,
  and a number like it survives a parse and re-serialise round trip, so a test
  that does not deliberately carry one will never see the difference.
- **Key order in nested objects** is the order stored, with one exception that
  comes from JavaScript and reaches every implementer: a key that looks like a
  non-negative integer sorts FIRST, in ascending numeric order, before every
  other key. A nested map keyed `10, 2, 1` serialises as `1, 2, 10`. A recorder
  building its records from its own data structures MUST reproduce that; an
  implementer re-serialising already-parsed JSON never sees it, because the
  reordering happened before the bytes were written.
- **Absent versus null.** For the four defaulted keys (`aiModel`, `aiProvider`,
  `detectionMode`, `configSnapshot`) a `null` is coerced to the default, so it
  hashes as the default does. For every other key `null` is a value and hashes as
  `null`, while `undefined` is omitted entirely. Until this paragraph existed that
  asymmetry was carried by the word "default" alone, and it is the likeliest
  thing to bite a second recorder.
- **Unicode normalisation** is NOT applied, in either direction. The bytes are
  hashed as the record carries them. A recorder that normalises its text before
  storing it and a checker that normalises before hashing would disagree with
  every file in the world, so neither does.

`vectors/11-text-escaping.html` carries an astral emoji, a lone high surrogate, a
curly quote, an accented letter, a `"`, a `\`, a `/`, all five short-form
controls, a `\u0001`, the number `1e-7` and a nested object with integer-like
keys out of order. An implementation that reproduces its `recordsDigest` has got
every rule above right. One that cannot has a bug that would otherwise appear as
a tampered record in somebody's evidence.

### A checker older than the file it reads

That rule protects old records under a new checker. The other direction has no
protection and cannot have one: a checker hashes the keys on its own list, so a
record written after a field was appended hashes to something the older checker
cannot reproduce. Its recomputation misses, and the only verdict it has is the
one that says the record was modified after signing.

That is the loudest possible wrong answer for this format, and it is not
hypothetical: six fields have been appended since this list was first written,
and the published checker is refreshed a week behind the extension.

A second implementer SHOULD therefore:

- state which version of this list the implementation was built against, so a
  reader can compare it with the file's own;
- when a recomputation misses, report the names of the keys on the record that
  are not on the list and not in the set outside the payload (`integrityHash`,
  `signature`, `chainAnchor`, `redacted`), and say plainly that a checker older
  than the file fails this way;
- NOT soften the verdict on that ground. A record whose content does not hash to
  its stored hash has failed. An unknown key is an explanation offered to a
  human, never a lighter result: anyone who can edit a record can also add a
  key, and a checker that downgraded on that would be trivially defeated.

The reference checker does this. An unknown key on an otherwise sound record
changes nothing there, because the payload only reads keys it knows; the names
are reported only when the recomputation has already failed.

## 4. The checks, and the three readings

For each record, in file order:

1. Withheld record: if the record carries `redacted`, its content is gone and
   its hash cannot be recomputed, so four other things are checked in its place.
   A `redaction-marker` in the same file must name the record's `id`; that
   marker's `recordHash` must equal the record's stored `integrityHash`; the
   record's own `redacted.markerId` must be that marker's id, so the two point
   at each other and not merely at the same hash; and the marker record must
   itself pass every check in this list. Any of those four failing gives
   `redaction-unattested`. Then, when the record carries a `signature` and the
   file carries a key to check it under, that signature is verified over the
   UTF-8 bytes of the stored `integrityHash`, exactly as for any other record,
   and one that does not verify fails the record with `tampered`. That signature
   is the only thing on a withheld record that says the install made it, because
   the content its hash covers has left the file. A withheld record carrying no
   signature is accepted on the marker alone and is not counted as `unsigned`,
   and one naming a key the file does not carry is `prior-key`, exactly as on
   the ordinary path: there is no key here to check it under, so the marker is
   all there is and the file reads partially verified. A checker that calls that
   record `tampered` reports BROKEN on a sound file from an install whose key
   has since rotated. This one did until 2026-09-18, on the withheld path only,
   while the ordinary path beside it read the same condition as `prior-key`.
   The note a checker prints MUST NOT say the signature of such a record was
   checked. There are three ways it goes unchecked and a note that counts only
   one of them makes that claim over the other two: the record carries no
   signature, the record names a key the file does not carry, or the file has
   no key of its own. This checker said "its signature over that hash was
   checked" over a record with none until 2026-09-19.
   `vectors/05-withheld-record.html` is a good marker,
   `vectors/20-withheld-unsigned.html` a withheld record with no signature at
   all, which is `verified` and not `unsigned`,
   `vectors/14-redaction-unattested.html` a marker that does not name the
   record, and `vectors/18-redaction-marker-mismatch.html` a record whose own
   `redacted.markerId` names a different marker than the one attesting it.
2. Hash: recompute the canonical payload and compare with `integrityHash`. A
   mismatch fails with `tampered`.
3. Signature: without a `signature` the record is `unsigned` (accepted, never
   green). Otherwise look up the key by `publicKeyFingerprint` (the file's own
   key when absent). A key not in the file gives `prior-key` (accepted by hash
   only). A signature that does not verify under the key the record NAMES fails
   with `key-mismatch`. A signature that verifies under a key that is not the
   file's own is `carriedKey` (accepted, never green: the file cannot say whose
   key it is).

   A record that names no key at all is read more softly, and a checker MUST do
   the same or it will disagree with this one. When `publicKeyFingerprint` is
   absent and the signature does not verify under the file's own key, the record
   is `signature-did-not-verify`: accepted, never green, counted under
   `unverified.signatureRejected`, and NOT a failure. The reason is that a
   record carrying no fingerprint may have been signed by a key that has since
   rotated, and nothing in the file can tell that apart from an edit; calling it
   BROKEN would assert an edit the file does not prove.

   The cost of that is worth stating plainly, because it is the sharpest edge in
   this document. On a file with no manifest, an edit to a record, with its
   unkeyed `integrityHash` recomputed and its old signature kept, reads `failed`
   with `key-mismatch` while `publicKeyFingerprint` is present, and reads
   `partial` with an empty `failures` list once that one field is deleted and
   the hash recomputed again. Deleting a field moves an edited file from BROKEN
   to PARTIALLY VERIFIED and the reference checker's exit code from 2 to 1. What
   stops this being silent is the counter and the sentence: `signatureRejected`
   is 1, and a checker MUST say that those records name no key, so the reading
   is a rotation or an edit and is not proof of either. A signed manifest closes
   the hole outright, because `recordsDigest` catches the edit before any of
   this is reached. `vectors/16-edited-unfingerprinted.html` is that file.

Then the chain: every `prevRecordHash` is resolved against every record in the
file. A predecessor present is a link. A predecessor absent where positions are
consecutive is a seam, counted as a restart. A position held twice is a branch
(counted). A predecessor absent where positions are not consecutive is a record
outside the file (counted). The first record in the file is no exception: it
has no record before it to be consecutive with, so when it sits above position 0
and its predecessor is absent it is one record outside the file, never a seam.
Until 2026-09-20 the reference checker skipped it, and a report whose ten
records each named a left-out predecessor said nine did. None of these fails a
file on its own; a modified record fails its own hash and a forged one its own
signature.

Positions are compared inside a run, and a run can be linked to the one before
it. Positions restart at 0 on purpose in two places, the month rotation and the
reset, and each writes a marker at position 0 whose `prevRecordHash` is the
constant `feelingwise-chain-genesis-v1` and whose `priorChainHash`, which is
inside the hashed payload, is the `integrityHash` of the last record of the run
it follows. A checker MUST start a new run at a `month-rotation-marker` or a
`reset-marker` at position 0 when two things hold: its `priorChainHash` is the
`integrityHash` of a record in the file, and the marker's own signature verified
under a key. Positions after such a marker are compared only with each other.
When either fails, nothing in the file shows which run the marker follows, and a
checker MUST count reused positions as branches exactly as it would without the
marker.

The record a linked marker names may already have a successor in the file. A
restore from a backup leaves exactly that: the backup puts back an older state
of the chain, the records written after the backup stay in the month archive and
so in every whole-record export, and the next reset or rotation names the record
the restore went back to. From that record the file then goes on two ways, the
new run and a line the record did not keep.

To count those lines, give every record one parent: for a linked marker, the
record its `priorChainHash` names; for any other record, the record its
`prevRecordHash` names, when that record is in the file. A record's children are
the records whose parent it is, so a linked marker goes on from the record it
names exactly as a successor does. A record that a linked marker names is a fork,
and at a fork one child carries on the line the record kept:

- The kept line is read back from the newest record in the file: the last record,
  in file order, whose `chainPosition` is a number of 0 or more and whose
  `chainAnchor` is not `pre-rekey`. Its parent, that record's parent, and so on
  back while each is in the file and none repeats, make the kept line. A
  whole-record export puts the archive first, then the live chain, then the
  records of earlier keys, so the kept line is the one the install is on now. At
  a fork on the kept line, the child on it is kept.
- At a fork the kept line does not pass through, the last linked marker in the
  file that names the fork is kept. That happens when a record missing from the
  file breaks the way back, and when the fork is itself on a line set aside.

A checker MUST count as `chain.setAside` every other child of every fork and
every record that goes on from each of them, child after child, each record once. It MUST count them before it
compares any position, and MUST NOT count them again as branches, whether the
file puts them before the marker or after it. It MUST report the count at each
fork in `chain.setAsideLines` as `{position, n}`, the fork's `chainPosition` and
the number of records counted there, one entry per fork with anything counted,
in the file order of the first linked marker naming each fork. A file with
anything set aside reads `partial`, and no `selection` excuses it: nothing signed
in the file says whether a restore made the line or somebody holding the key
replaced the newest records and signed the replacements, and the note MUST say
that it cannot tell which. This is also why linking to a record in the middle of
a run erases nothing: the named record then has at least two children and only
one is kept, so the other line is still counted, as set aside instead of as
branches.

Four layouts show the rule, each read from signed records by this checker. A
backup at position 5, a reset, three records, the backup restored and a reset
again: 4 set aside from position 5, the first reset and its three records. A
backup at position 5, two records, a reset naming position 7, three records, the
backup restored and a reset: 6 set aside from position 5. Two restores to one
backup at position 5 with two records written between them, then a reset: 4 set
aside and no branch. Backups at positions 5 and 7, the first restored, a reset
and four records, then the second restored and a reset: 5 set aside from
position 5, the first reset and its four records, however long recording then
goes on, because the record went on from position 7.

The file's order decides which line was kept, because nothing signed says so. A
file that puts a side line after the live chain is read as keeping that line.

The genesis constant is not a record, so a `month-rotation-marker` or a
`reset-marker` at position 0 is never counted as naming a predecessor outside
the file for its `prevRecordHash`, whether or not it links. What such a marker
follows is its `priorChainHash`: when that is a non-empty value that names no
record in the file, the marker counts as one record outside the file. The first
record in a file is the exception it always was, and names nothing.

`vectors/21-month-rotation-linked.html` reads `verified`.
`22-month-rotation-unlinked.html` (the record named is not in the file) reads
`partial` with three branches and one record outside.
`23-month-rotation-names-a-middle-record.html` names the first record of the
month before, so it links and reads `partial` with two records set aside from
position 0, no branch and nothing outside. `24-month-rotation-marker-unsigned.html`
names the right record on a marker with no signature, so it does not link, and
reads `partial` with three branches and nothing outside. Until 2026-09-20 the
reference checker did not read the link, and every whole-record file that
crossed a month boundary read PARTIALLY VERIFIED. Until 2026-09-23 it also
refused to link a marker whose named record had a successor, so a whole-record
export made after a restore and a reset counted every position the two runs
shared, 8 in a file of 21 records made by the product's own buttons, and named
the reset marker's genesis constant as a missing predecessor.

A cut before a reset cannot be counted the same way. When the records at the end
of the chain a reset retired are missing, the record the marker names is missing
too, and the file carries no position for a record it does not hold, so how many
were cut cannot be read from it. It reads `partial` with that one record
outside, and the reused positions count as branches because nothing links.

An archived record the install could not read leaves the same hole a removal
leaves: the record after it names a predecessor outside the file, and its
position is missing from its run. When a checker has taken the file's
`unreadableArchivedRows` (section 1) and the records outside the file are no
more than that count, its note MUST name that cause, as the install's statement
under its signature, and MUST NOT offer a filtered extract or a removal as the
explanation. The reading stays `partial`: the count explains the holes and does
not fill them, and nothing signed says which records they were. When the record
that could not be read is the one a later marker names, the marker does not
link, so the positions after it are compared with the run before it as for any
unlinked marker.

A seam is also either attributed or not, and the rule is narrower than it
sounds. It is attributed only when the records on BOTH sides of it came back
green: each one passed, and neither was marked unverified. Exactly four readings
mark a record unverified, and any one of them on either side leaves the seam
unattributed and increments `chain.unattributed` beside `chain.restarts`:
`unsigned`, `prior-key`, `signature-unverified` and `signature-did-not-verify`.

`carriedKey` is not one of them, which is the half an implementer is likely to
get wrong in the other direction. A record that verified under a key this file
carried beside the records, rather than under the file's own key, is not marked
unverified: its signature did verify, and a seam beside it is attributed. What
the file cannot say about that record is whose key it was, which is a different
thing and is counted separately.

What attribution distinguishes is worth saying out loud. An attributed seam is
an install that lost records and carried on signing with a key this file holds,
so the same hand is on both sides of the gap. An unattributed one is a gap the
file cannot speak for, most often because the generation before it was signed by
a key that has since been replaced and is not here. Neither is a failure and
neither is evidence of one. A checker that attributes on the link alone, or on
the two records merely passing their hashes, will report a seam as accounted for
that this one does not. `vectors/15-seam-and-branch.html` carries an attributed
seam beside a branch, `vectors/19-unattributed-seam.html` an unattributed one.

Then the anchors: a time-stamped record hash absent from the file, or a
time-stamped position past the last record, is counted (section 6).

The reading:

- **failed** (BROKEN): any record failed, or the manifest failed, or the file's
  own key could not be set up. The third is checked before everything else and
  is terminal the same way a manifest failure is: `records: 0`, `keys: 0` and a
  single failure with `index: -1` and `reason: key-setup`. It has three causes,
  told apart by `detail`: the `publicKeyJwk` carries no string `x` and `y`, the
  fingerprint recomputed from those coordinates is not the stated
  `publicKeyFingerprint`, or the key is well formed as JSON and is not
  importable as a P-256 public key, where `detail` is whatever the crypto
  library said. `vectors/17-key-setup-fingerprint-mismatch.html` is the second
  of those three.
- **partial** (PARTIALLY VERIFIED): nothing failed but something is `unsigned`,
  `prior-key`, `signature-did-not-verify` or `carriedKey`; or there is a seam, a
  branch or a record outside the file that the file's own selection does not
  account for; or any record is set aside; or an anchor names something not
  here; or there is no manifest.
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
  `<dt data-fw-prov="name">`, every `<dd data-fw-prov="name">` AND every
  `<p data-fw-prov="name">` element in the
  HTML, as `[name, text]` pairs sorted by name, `text` being the element's text
  content with tags dropped, entities decoded, whitespace collapsed and trimmed.
  This is why a bare JSON envelope cannot fully verify. A checker reading one
  MUST report that comparison as not made and cap its verdict at partially
  verified, and it must not report it as failed. A bare envelope has no page rows,
  so a digest over zero rows can never equal the manifest's digest over the
  page's rows, and treating that as a mismatch tells a recipient a sound file is
  broken. Everything else in the manifest still applies: `recordsDigest` and
  `envelopeDigest` are checked before this step, so a record removed from or
  reordered in a bare envelope still fails. Since 2026-09-19 the manifest's own
  `digest` and its signature are checked before this step as well. Without that,
  a bare envelope was never signature-checked at all while the report still
  called the manifest checked, and every digest in it recomputes for an editor
  who recomputed them.

  Those three tags, and only those three. The summary rows are `<dd>` inside the
  definition list; the labels that introduce them are `<dt>`; the disclosure
  block and the paragraph naming the limits of the check are `<p>`. A checker
  matching `<dd>` alone leaves those paragraphs outside the signature, and one
  matching `<dd>` and `<p>` leaves every label outside it, which was true of
  this format until 2026-09-18: the signed digits of the signing key fingerprint
  could be presented under the words "Archive fingerprint" and the file still
  read VERIFIED. A checker matching *any* tag would disagree with the
  recorder the first time a fourth kind of element carries the attribute, so the
  set is named explicitly here and in all three shipped implementations.

  A checker that reads the HTML with a pattern rather than with a DOM MUST close
  each row on that row's OWN closing tag. Closing on whichever of `</dd>` and
  `</p>` comes first was harmless while only those two carried the attribute and
  became wrong the moment `<dt>` did, because a label is immediately followed by
  the value it introduces: such a checker signs the label as though it were the
  value. That is the shape of every defect this digest has had, and it is always
  the same shape, which is that the three implementations of one reading rule
  drifted apart.

  The
  attribute MAY sit anywhere in the tag: a browser's `querySelectorAll` does not
  care about attribute order, so an implementation that reads the HTML with a
  pattern MUST NOT require the attribute to come first. Anchoring it there is
  how one shipped implementation came to read fourteen rows where the page read
  sixteen, and every file built that day read BROKEN on this digest.

  The `name` is letters and dots, `[a-zA-Z.]+`, and MUST be unique within a
  file: the pairs are sorted by name alone, so two rows sharing one would sort
  unpredictably between implementations. A label's name is the name of the row
  it introduces with `.label` after it, so the two sort next to each other and
  neither can be presented as the other.

  WHICH ROWS A RECORDER MARKS. Everything the page states about the file or
  about the records in it, and nothing that is decoration. In the reference
  recorder that is: the record count, the flagged counts and the flagged share,
  the unattended count, the builds, what was held back and what it was drawn
  from, the chain range and every coverage row, the count of archived records
  the install could not read, the recorded date range, the
  signing key fingerprint, the conditions, the per-key table and the count of
  keys above it, why the key exists, when the key was made, the build that
  recorded it, the maker and the machine, the precision disclosure, the
  paragraph naming what the file reveals about its owner, and the paragraph
  naming what the check does not establish, and since 2026-09-18 the `<dt>`
  label that introduces each of those rows. The title and the subtitle are not
  marked: they name the product, not the file.

  A LABEL IS COVERED SINCE 2026-09-18, and a second implementer reading an older
  file should know what changed. Before that date the digest read `<dd>` and
  `<p>` elements only, a `<dt>` was neither, and an editor could present the
  signed digits of the signing key fingerprint under the words "Archive
  fingerprint" with the file still reading VERIFIED. That was measured, recorded
  as the one edit of twenty four a tampering corpus did not refuse, and closed by
  marking each label as its own row rather than by widening what a checker reads
  without being told. A file exported before that date carries no marked labels,
  so the rule below applies to it and it still reads VERIFIED.

  A CHECKER MUST READ THE MARKED SET THE FILE CARRIES, never a list of its own,
  and this is what lets a recorder mark more rows later without breaking files
  already in people's hands. Until 2026-09-18 the reference recorder marked seven
  rows and the rest of the summary list was under no digest at all: the coverage
  sentence, the key fingerprint and the limits paragraph could each be rewritten
  in a text editor with the file still reading VERIFIED, manifest ok. Widening
  what the recorder marks changes nothing for a file signed before the widening,
  because its manifest digested exactly the rows it carries and the same reader
  finds exactly those. Widening what the CHECKER reads, to unmarked `<dd>`
  elements say, would refuse every file already exported, which is worse than
  the gap it would close.

  Entities decoded, in this order: `&#39;` `&quot;` `&lt;` `&gt;` `&amp;`.
  `&amp;` must be decoded LAST, or `&amp;lt;` collapses to `<` instead of
  `&lt;`.

  Once these rows are known to be the ones the key signed, and not before, a
  checker compares the envelope's `profile` against the `madeBy` and `machine`
  rows among them and refuses a file where they disagree (section 1).
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

THE PINNED AUTHORITIES, which until 2026-09-18 were readable only inside the
reference implementation. An independent checker written from this document
alone had to guess them, and a wrong guess is the worst outcome this format has:
a file genuinely dated by DigiCert reads "dates nothing on its own" in one lab's
checker and "dated by DigiCert" in another's, on the one fact a court would care
about. The map is root certificate SHA-256, lowercase hex, to the exact display
name a report prints:

| Root certificate SHA-256 | Display name | Certificate |
|---|---|---|
| `a6379e7cecc05faa3cbf076013d745e327bbbaa38c0b9af22469d4701d18aabc` | `FreeTSA (freetsa.org) Root CA` | O=Free TSA, OU=Root CA, CN=www.freetsa.org, self-signed, 2016-03-13 to 2041-03-07 |
| `c0712e7f295d643e9aa6b72686dde19ab68169c22b63a13bf91580d9055262ab` | `CESNET CA Root` | DC=cz, O=CESNET CA, CN=CESNET CA Root, self-signed |
| `33846b545a49c9be4903c60e01713c1bd4e4ef31ea65cd95d69e62794f30b941` | `DigiCert Trusted Root G4` | C=US, O=DigiCert Inc, CN=DigiCert Trusted Root G4 |
| `b53ac15cc1afb6e2ac06828f555bb3bf5bad8b2bac1733ce4cb7aafe729356de` | `Sectigo Public Time Stamping Root R46` | C=GB, O=Sectigo Limited, CN=Sectigo Public Time Stamping Root R46 |

A checker MAY pin a different set, and SHOULD say which set it used. What it
MUST NOT do is treat an unpinned root as a failure of the file: the token either
verifies or it does not, and whether the authority at the top of it is known is a
separate result reported beside it. Those two are independent in the reference
implementation and a report that blends them tells a reader a sound token is
broken. Pinning is a statement about who the checker is willing to believe, not
about whether the file is sound.

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

The largest of those absences, that a post existed on the platform, is the one
section 9 is about. It cannot be closed inside a single file by any means at all,
and what section 9 provides is the one thing that makes it approachable across
many files: a content identity, so that two strangers who recorded the same item
give it the same name.

## 8. What a report says, which is part of the format

Added 2026-09-18, after an engineer wrote an independent checker from this
document alone and got nine of the ten vectors that existed that morning right.
Everything they could not get is here. `verifier/vectors/EXPECTED.json` is the
comparison target this folder invites an implementer to diff against, so its
fields are normative and were defined nowhere; an implementer had to reverse the
comparison target from the comparison target.

EXPECTED.json is a PROJECTION of a report and not a report. It carries a fixed
list of fields, flattens some of them out of the `chain` and `timestamps`
objects (`outside`, `branches`, `restarts`, `unattributed`, `setAside`,
`anchorsOk`, `anchorAuthorities`, `keyDatedBy`), and normalises an absent
`holding` to `null`. Diff your report's corresponding fields against it, not
your whole report.

### The manifest status

One token, the first check that fails, in this order. A checker MAY use other
names internally and MUST be able to say which of these conditions it found.

| Token | The condition |
|---|---|
| `missing` | no manifest object at all |
| `legacy` | `version: 1`, the files of one morning in September 2026; treated as absent |
| `version` | a `version` that is neither 1 nor 2 |
| `key` | the manifest's `publicKeyFingerprint` is not the envelope's |
| `records` | `recordCount` is not the number of records, or `recordsDigest` misses |
| `anchors` | `anchorsDigest` misses |
| `keys` | `publicKeysDigest` misses |
| `labels` | `techLabelsDigest` misses |
| `wording` | `i18nDigest` misses |
| `envelope` | `envelopeDigest` misses |
| `digest` | the manifest's own `digest` misses |
| `signature` | the manifest's signature does not verify |
| `no-rows` | there is no page to read rows from: a bare JSON envelope. NOT a failure |
| `page-rows` | `provenanceDigest` misses |
| `profile` | the envelope's `profile` disagrees with the signed rows rendered from it |
| `ok` | every check above passed |

`digest` and `signature` sit ABOVE `no-rows` on purpose, and moved there on
2026-09-19. Every digest above them is computed from the file's own data, so a
file that was edited and then recomputed is self-consistent; the manifest
signature is the only thing binding the manifest to the key that signed it. A
checker that lets a bare JSON envelope decline the row comparison first never
reaches the signature at all, and "remove some records and recompute" passes
clean on the one input form this document tells you to accept. Order them as
the table does.

`envelope` and `page-rows` were called `provenance` and `summary-rows` until
2026-09-18, which had them exactly the wrong way round: the token for a failed
`envelopeDigest` was the word "provenance", and the token for a failed
`provenanceDigest` was not. An implementer mapping this document's field names
onto status tokens produced the inverted pair and then told a reader the page
rows had been edited when the envelope had. Renamed while no third party had yet
built on them.

### A manifest failure is terminal

A checker that finds the manifest bad MUST NOT go on to report on the records.
It reports `records: 0`, `keys: 0`, `holding: null` and a single failure with
`index: -1`, and its verdict is BROKEN. `holding: null` means the field is
present and null: a reader cannot tell an absent key from a field a checker
forgot, and this is the one place where what the file is holding genuinely
cannot be read. The reason this is not a matter of taste is section 5's own:
the verdict wording lives under the manifest, so a checker
that keeps reporting is reporting through prose it has just found unsigned. Two
checkers that disagree here agree on BROKEN and differ on four machine-readable
fields, which is what a lab diffing them sees.

### The failure reasons

`index` is the record's position in the file, or `-1` for a failure that is not
about a record. `reason` is one of: `tampered`, `key-mismatch`,
`redaction-unattested`, `manifest` (with the manifest token in `detail`), and
`key-setup` (the file's own key could not be set up, with the cause in `detail`;
section 1 calls this refusing the file, and it reads BROKEN, not unreadable).

`unsigned`, `prior-key`, `signature-did-not-verify` and `signature-unverified`
are readings of a record and never failures. They are counted under `unverified`
and cap the file at partially verified; a checker that puts them in `failures`
reports BROKEN where this one reports PARTIALLY VERIFIED. This document listed
the first two as failure reasons until 2026-09-18 and they had never been
emitted as any.

Two of the five reasons cover more than one condition. `tampered` is both a
recomputed payload that does not match the stored `integrityHash` and a withheld
record whose signature over that stored hash does not verify.
`redaction-unattested` covers the four conditions listed in section 4.

The fourth reading, `signature-unverified`, counted under
`unverified.signatureUnverified`, is a record carrying a signature in a file
that offers no key at all to check it under. The reference checker cannot reach
it: it refuses a file whose own key will not set up, before any record is read,
so that counter is zero in every reading it produces. A checker built to carry
on past a key it could not import will reach it, and this is the token for it.

### The report's other fields

`state` is `verified`, `partial` or `failed`. `records` and `keys` are counts.
`carriedKey` counts records signed by a key that travelled in the file rather
than the file's own. `redacted` counts records whose content was withheld on a
signed marker. `unverified` is `{priorKey, unsigned, signatureUnverified,
signatureRejected}`. `coveredRows` names, in sorted order, the page rows the
signature covered, which is a fact about the FILE and not about the checker: an
older file marks fewer. `holding` is the four fields EXPECTED.json
compares, `{kind, included, totalSource, leftOut}`, where `kind` is `report`
when the envelope carries a `selection` and `whole-record` otherwise. The
reference checker's own `holding` object carries eight more beside them, taken
from the envelope and the manifest for the opening summary it prints:
`records`, `generatedAt`, `fwVersion`, `fingerprint`, `anchors`,
`keyStamped`, `manifestOk` and `unreadable`, the count of section 1's
`unreadableArchivedRows` when the checker took it and 0 otherwise. Those eight
are convenience and not contract; a checker MAY leave every one of them out. `anchorsOk` and `anchorAuthorities`
are independent:
a token can be sound under an authority the checker does not pin, and a report
that blends the two tells a reader a sound token is broken. `keyDatedBy` is the
display names of the authorities whose verified tokens over the signing key the
checker accepted, in the order it met them, or null when the file carries no
`keyGenesis` anchors. It is a list of NAMES and not a time, which its own name
invites an implementer to get wrong: the time is
`timestamps.keyGenesis.existedBy`, the earliest `genTime` among those same
tokens.

### The payload element

Section 1 shows the opening tag of `fw-export-data` in a code block. That is what
a recorder writes. What a checker READS is any `<script>` element whose
attributes include `id="fw-export-data"`, in any order, with any whitespace
between them, because that is what a browser's `getElementById` sees. Matching
the literal string is how, until 2026-09-18, a conforming file that wrote `id`
before `type` was answered with "not a FeelingWise export", which is the worst
thing this checker can say about a sound file. The same rule, and the same
reason, as the attribute-position paragraph in section 5.

## 9. Content identity, and why the format has one

Added 2026-09-21. Vectors: `verifier/vectors/CONTENT-IDENTITY.json`.

### What it is for, which matters more than the rule itself

Section 7 says what a VERIFIED file does not claim, and the largest of those
absences is this: the format cannot show that anything was ever served. A record
signed over a fabricated post verifies perfectly. The signature is applied after
the text is in hand, so no amount of cryptography reaches back past it.

The only thing that does reach it is other people. An observation earns weight in
proportion to how many unrelated observers recorded the same item being served,
and forging at scale costs a forger in proportion to how many independent
recorders they would have to control. That is not a property of a file. It is a
property of a corpus, and it is unavailable unless one served item gets one name
from two strangers who have never met.

Before this section it did not. Each record carries `originalHash`, a SHA-256 of
`originalText` exactly as extracted, and two independent extractions of one post
differ constantly: composed against decomposed accents, a non-breaking space
where markup had `&nbsp;`, a line ending, a stray zero-width character, the
layout whitespace one extractor keeps and another drops. Measured against eight
such differences, one agreed.

### The rule

A content identity is derived, never stored. It is computed from `originalText`,
which is inside the canonical payload and therefore already covered by the
record's signature (section 3), so a reader who has verified a record can compute
this and needs nothing else from us. Nothing in the record changes to carry it,
no existing signature is affected, and it applies to every record ever written.

Given `originalText`, the NORMAL FORM is produced by applying, in this order:

1. Unicode normalisation to **NFC**.
2. Every `\r\n` and every lone `\r` becomes `\n`.
3. These characters are removed: U+200B ZERO WIDTH SPACE, U+FEFF BYTE ORDER
   MARK, U+00AD SOFT HYPHEN, U+180E MONGOLIAN VOWEL SEPARATOR.
4. These become a single U+0020 SPACE: U+0009 TAB, U+00A0 NO-BREAK SPACE,
   U+1680, U+2000 through U+200A, U+202F, U+205F, U+3000.
5. Runs of two or more U+0020 become one.
6. Each line is trimmed of leading and trailing spaces; runs of three or more
   newlines become two; the whole string is trimmed.

The identity is then the literal string `fw-ci-1:` followed by the **lowercase
hex SHA-256 of the UTF-8 bytes of the normal form**.

The version lives inside the string and not in a neighbouring field on purpose.
Identities travel: into a spreadsheet column, into somebody else's database.
A bare digest from this rule and one from a later rule would compare unequal with
nothing to say the two were computed differently, and a silent false NON-match is
how a corroboration count quietly becomes wrong.

### What an implementer must NOT add, and the reason is one sentence

**A false merge is worse than a false split.** Two different items collapsing into
one identity manufactures corroboration that never happened, which is exactly the
fabrication this whole mechanism exists to resist. Two records of one item failing
to meet costs a little evidence and lies to nobody.

So the normal form removes only what a text extractor can add or drop by itself,
and never touches content. In particular a conforming implementation does not
fold case, does not unify quotation marks or dashes, does not strip punctuation,
and does not fold emoji to a base form.

It also KEEPS U+200D ZERO WIDTH JOINER and U+200C ZERO WIDTH NON-JOINER, which
look like the invisible noise removed in step 3 and are not. The joiner is what
makes one family emoji out of three separate people, and both characters shape
letters in Arabic, Persian and the Indic scripts. Removing them would merge texts
that genuinely differ, and would do it hardest in the languages least represented
in any corpus assembled so far.

### Two things this rule refuses to repair, and both are correct

**A caption the page only partly carried.** A text cut at a "show more" control is
less text, not the same text worn differently, and the cut point moves with the
viewport, so two observers of one post hold two different strings and are each
right. Normalising them into agreement would be a false merge, and a systematic
one. `ForensicRecord.textCapture` says which case a record is in: `whole` and
`folded` mean the page carried the item's whole text and the identity may be
compared; `cut` means it did not; `platform` means the text came from the
platform's own payload rather than the screen, and may corroborate other
`platform` captures but not what a person's screen showed. The field is absent on
records written before 2026-09-05, and **absent is not `whole`**: those records
answer "unknown" and a corroboration count must say so rather than lean on
history it cannot vouch for.

**A joiner one extractor dropped.** Reported rather than repaired, for the reason
above.

### What an identity does not claim

That two records carry one identity means two files contain the same text. It
does not mean either observer saw it on a platform, it does not date either
observation, and it is not by itself evidence that anything was served. It is one
input to a corroboration judgement, and everything in section 7 still holds.

Identical short text from different people produces one identity, by definition of
content addressing. A corroboration claim resting on a few words, or on a single
emoji, is weak evidence and an implementation should make the length visible
rather than pretend otherwise.

### Conforming

`verifier/vectors/CONTENT-IDENTITY.json` holds 25 vectors in 12 groups. Every
vector sharing a `group` must produce the identity given; any two groups must
produce different identities, with one exception the file declares, which is that
`empty` and `whitespace-only` both normalise to the empty string. The groups
ending `-cut` and `-unjoined` are the half that must NOT merge, and an
implementation that merges them is wrong in the direction that matters.

The file is generated from the shipped rule by
`scripts/build-content-identity-vectors.ts`, so it cannot drift from the code by
being edited or by being forgotten. Regenerate it with `npm run gen:ci-vectors`.

`corroborate.mjs`, beside this document, is a working implementation of this
section in about eighty lines and is the shortest way to read the rule as code.
It is checked against the vectors above and against the extension's own copy, so
the two cannot say different things. What it does with the identities afterwards
is described in the README, and the part worth copying is what it REFUSES to do:
two exports signed by one key are one observer, and counting their overlap as
agreement is the easiest way to produce a number that means nothing.

`observations.mjs` uses the same rule for a different job: it writes out what a
file says was served, with every model reading left out, and puts the identity
beside each row. Between them they are the two things section 9 exists for, which
are comparing one file with another and citing one file on its own.
