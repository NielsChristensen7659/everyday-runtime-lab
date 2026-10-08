# PDF Compression Explained Image Quality Trades for Signed Game Contract Storage

Compress game contract PDFs before signing them, then preserve the signed bytes unchanged. The deciding constraint is the audit trail: a smaller archive is useful, but a signature over one byte sequence cannot vouch for a different, recompressed sequence. For scanned exhibits, compression mainly trades away image information, so judge it by evidence legibility rather than by file size alone.

TL;DR: normalize and compress unsigned inputs, inspect representative pages, hash the final PDF, sign that exact artifact, and store the original hash plus every custody event. Do not put post-signing optimization in the archive path. Treat signatures, timestamps, and audit records as evidence with retention rules of their own.

## What Image Quality Does PDF Compression Trade Away?

A PDF is a container, not one big picture. It can hold text, fonts, vector paths, metadata, and several image streams. Those parts respond differently to optimization. Removing duplicate resources or compressing an uncompressed stream can reduce bytes without changing rendered content. Resampling a 600 dpi scan to 200 dpi cannot. Lossy image encoding discards detail to make the image stream smaller.

That distinction gets blurred by a single UI slider labeled “quality.” I do not trust that slider. I want the resulting pixel dimensions, encoding, color handling, and file size in the build log. Configuration bloat starts when an archive policy becomes a pile of opaque presets.

The damage is rarely evenly distributed. A full-page screenshot may still look fine while a faint signature, a one-pixel checkbox, or the small print in a licensing exhibit becomes ambiguous. Chroma subsampling can also soften colored edges more than black text. If optical character recognition produced a hidden text layer, readable search results do not prove that the visible scan remains adequate evidence.

PDF compression therefore trades among four things: spatial detail, tonal or color detail, future reprocessing headroom, and storage or transfer size. The first three are hard to recover. Storage can be bought later.

Pixels do not grow back.

## Why does signing change the build order?

A digital signature binds a signature value to a defined byte range in the PDF. PDF also permits incremental updates, and PAdES defines profiles for PDF signatures, including forms intended for long-term validation. Those details matter because “the document still opens” is a poor signature test. A rewrite may preserve the apparent page while changing the signed bytes or the validation state.

The clean pipeline is short:

1. Receive the unsigned contract and attachments.
2. Normalize allowed inputs and apply the archive compression policy.
3. Run visual and structural checks on the candidate.
4. Hash and sign that exact candidate server-side.
5. Store the signed artifact as immutable content, with a separate append-only custody log.

No victory lap yet. The signature says nothing about whether a human could read the compressed signature image before signing. That is why visual acceptance belongs before the cryptographic boundary.

For a game studio, the test set should reflect the actual deal packet: typed publishing terms, scanned talent releases, handwritten initials, royalty tables, territory maps, and image-heavy art schedules. Sample by page class rather than taking the first ten pages. The ugly pages set the policy.

## The smallest auditable server-side boundary

The compressor and signer can be separate services. Keep their contract boring: bytes in, bytes out, plus explicit evidence. This TypeScript sketch avoids prescribing a library or provider and makes the irreversible boundary visible.

```ts
import { createHash } from "node:crypto";

type Artifact = {
  bytes: Uint8Array;
  sha256: string;
  byteLength: number;
};

type AuditEvent = {
  contractId: string;
  action: "compressed" | "approved" | "signed" | "archived";
  artifactSha256: string;
  occurredAt: string;
  actor: string;
};

type PdfCompressor = (source: Uint8Array) => Promise<Uint8Array>;
type PdfSigner = (approvedPdf: Uint8Array) => Promise<Uint8Array>;
type AppendAudit = (event: AuditEvent) => Promise<void>;

const describe = (bytes: Uint8Array): Artifact => ({
  bytes,
  sha256: createHash("sha256").update(bytes).digest("hex"),
  byteLength: bytes.byteLength,
});

export async function archiveContract(
  contractId: string,
  source: Uint8Array,
  actor: string,
  compress: PdfCompressor,
  sign: PdfSigner,
  appendAudit: AppendAudit,
): Promise<Artifact> {
  const candidate = describe(await compress(source));
  await appendAudit({
    contractId,
    action: "compressed",
    artifactSha256: candidate.sha256,
    occurredAt: new Date().toISOString(),
    actor,
  });

  // A real workflow gates this call on visual and structural approval.
  await appendAudit({
    contractId,
    action: "approved",
    artifactSha256: candidate.sha256,
    occurredAt: new Date().toISOString(),
    actor,
  });

  const signed = describe(await sign(candidate.bytes));
  await appendAudit({
    contractId,
    action: "signed",
    artifactSha256: signed.sha256,
    occurredAt: new Date().toISOString(),
    actor,
  });

  return signed;
}
```

The example deliberately does not claim that hashing signs a file. SHA-256 produces the content identifier used by the log; the signing component must implement the chosen signature profile, certificate handling, and validation policy. It should return a PDF whose signature is then independently validated before archival.

I would benchmark this boundary with a fixed corpus, not a synthetic blank document. Record input bytes, output bytes, elapsed time, peak memory, page count, validation result, and the policy version. A median hides the page that takes 30 seconds or exhausts memory, so retain tail latency and the largest expansion too. Some already-compressed files can grow after another pass. Rejecting growth is a reasonable policy, but only before signing.

## How should image quality be accepted?

Start with a reversible baseline. Keep the source during policy development, render source and candidate with the same conforming renderer, and compare the same pages at the same scale. Automated pixel metrics can catch broad regressions, but they do not know that a faint initial matters more than a background texture.

Use a small acceptance matrix instead of one magic score:

| Page class | Failure to look for | Practical gate |
| --- | --- | --- |
| Born-digital terms | Font substitution, missing glyphs, shifted line breaks | Text extraction plus rendered-page comparison |
| Scanned signatures | Broken strokes, merged initials, lost faint ink | Human review at normal and zoomed viewing sizes |
| Royalty tables | Decimal points, grid lines, superscripts | Targeted crops and field-level review |
| Color exhibits | Changed labels or indistinguishable legend colors | Color-aware visual comparison |

Set numerical thresholds from labeled examples your legal and records teams have accepted. Do not borrow a dots-per-inch number from a blog and call it compliance. PDF page dimensions and image pixel dimensions let you calculate effective resolution, but acceptable resolution depends on the smallest meaningful feature in the source.

Test failure handling too. A timeout must not silently fall back to the unapproved source, and a malformed attachment must not produce a partially signed packet. Each attempt needs a stable contract identifier, an input hash, a policy version, and a terminal status. Retries should be idempotent at the workflow layer so a network interruption does not create two independently signed “final” artifacts.

This is where developer experience matters. One typed policy object, one corpus runner, and one report beat fifteen environment flags. I would fail deployment when the corpus loses pages, changes extracted contract identifiers, fails signature validation, or crosses agreed visual gates. File size is reported. It is not the sole pass condition.

## What I would change at archive scale

At low volume, storing both the source and approved signed artifact makes investigation easier. At archive scale, retention, access control, and duplication deserve explicit decisions. Content-addressed storage can deduplicate identical artifacts, but deduplication keys and metadata can leak relationships if access boundaries are sloppy. Encrypt stored objects, restrict audit-log writes, and keep verification separate from the service that produced the signature. I would also move rendering and compression into isolated workers with strict CPU, memory, input-size, and execution-time limits. PDFs can contain complex objects and active features, so the archive path should reject content outside its policy rather than letting a general-purpose processor roam across the application host. Long-term validation is a lifecycle, not a checkbox at upload time: certificates expire or are revoked, algorithms age, and validation material may need preservation. PAdES long-term profiles and the organization’s records policy should drive periodic validation and evidence renewal. Recompressing the signed contract is still the wrong lever.

Freeze means freeze.

The trade-off is operational weight. Dual storage, sandboxed workers, independent validation, and append-only logs cost more engineering effort than calling an optimizer and saving its output. They also make the system explainable: the team can show which bytes were approved, which bytes were signed, which policy produced them, and who moved them into the archive.

My decision rule is blunt: optimize unsigned pixels until the worst meaningful exhibit still passes review, then freeze the artifact and protect its proof. A tiny PDF with questionable initials is not an optimization. It is missing evidence.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- ETSI EN 319 142-1, PAdES digital signatures: https://www.etsi.org/deliver/etsi_en/319100_319199/31914201/01.01.01_60/en_31914201v010101p.pdf
- NIST FIPS 180-4, Secure Hash Standard: https://csrc.nist.gov/pubs/fips/180-4/upd1/final
- OWASP File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
