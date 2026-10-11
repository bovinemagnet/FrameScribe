# Local Photo Catalogue — Product Requirements Document

Version: 1.0 · Date: 11 October 2026 · Status: proposed implementation baseline

## 1. Product definition

Build a private, local application that catalogues an existing photo collection in place, understands visual content, supports natural-language and similar-image search, and writes portable descriptive metadata into XMP sidecar files beside the images. Originals must remain byte-for-byte unchanged.

The primary user is a technically experienced photographer with JPEG and RAW collections, including Nikon NEF and Fujifilm RAF. The application should work from a CLI on a workstation or headless server and provide a small local web interface for browsing, reviewing metadata, and searching. It must not require a cloud account, paid inference API, or moving photos into a proprietary library.

Working product name: **Local Photo Catalogue**; example executable: `photocat`.

### Recommended implementation

Use Java 21 or 25, Quarkus, Gradle, a local SQLite catalogue, ExifTool for metadata extraction, and a LibRaw-based preview adapter. Use direct HTTP clients to independently configured embedding and captioning services. Start with a JVM distribution; native compilation is optional and must not delay the first useful release.

The application is a deterministic processing pipeline. An autonomous agent framework is unnecessary: models return bounded data, while application code owns traversal, decisions, validation, and writes.

## 2. Verified model basis and unresolved integration questions

The intended Google model is **EmbeddingGemma 2**, distinct from the original text-focused EmbeddingGemma, the Gemma 2 generative model family, and the hosted Gemini Embedding 2 product. Google's model card describes local multimodal embeddings, native 768-dimensional output, and smaller supported dimensions. Use 768 dimensions initially and include only the text and vision capabilities needed by this application. [S1]

An embedding is a numeric representation, not a caption. The product therefore has two independent AI capabilities:

| Capability | Model role | Product output |
|---|---|---|
| Visual retrieval | EmbeddingGemma 2 embeds images and text queries in a compatible space | Search rankings, similar images, optional controlled-vocabulary tag suggestions |
| Metadata generation | A separate local vision-language model | Proposed title, description, keywords and optional visible text |

Ollama lists `embeddinggemma-2`, including a `440m` text-and-image variant. However, the published `/api/embed` reference reviewed for this PRD documents text inputs, and the catalogue page alone does not prove image embedding works through the installed API on each OS/backend. Its displayed context-window metadata also differs from Google's model card. **Do not invent an image request schema or treat the model listing as a completed integration test.** [S2, S3]

Phase 0 must establish a tested runtime/model/version/platform matrix. Preferred transport is Ollama if actual image-to-vector inference passes. Otherwise run EmbeddingGemma 2 using its supported local model stack behind a small HTTP service. A Python inference worker is acceptable; the main application remains Java, Go or Rust. Pin worker dependencies and model revisions, and follow the model's documented processor, pooling, normalisation and task-prefix conventions. [S4]

For captions, evaluate a locally available Gemma vision model through Ollama. Ollama documents vision requests with structured output; JSON schema conformance still needs application validation and does not establish factual accuracy. Select the precise model and quantisation through a photo benchmark rather than assuming a large model is necessary. [S5]

## 3. Goals and non-goals

### Release 1 goals

- Index existing directories incrementally without reorganising images.
- Extract trustworthy technical metadata and keep it distinct from AI interpretation.
- Generate useful, editable descriptive metadata and adjacent XMP sidecars.
- Find photos using descriptions such as “a swimmer diving into an outdoor pool”.
- Find visually related images using another image as the query.
- Resume interrupted work and avoid repeating expensive inference unnecessarily.
- Preserve existing titles, keywords, ratings, copyright and editing instructions.
- Operate offline after models and dependencies have been installed.
- Support local disks and mounted NAS photo directories, with the catalogue on local storage.

### Excluded from Release 1

Photo editing, RAW development UI, cloud sync, mobile apps, face identification, automatic naming of people, inferred sensitive traits, automatic deletion of duplicates, video/audio indexing, autonomous external actions, and multi-user concurrent catalogue writers. Similarity search is not a face-recognition feature.

## 4. Scope and user workflows

| Priority | Workflow | Required result |
|---|---|---|
| P0 | Add one or more photo roots | Validate access, show supported formats, preview sidecar naming and detect collisions |
| P0 | Scan a root | Record files, technical metadata and processing status; show unreadable files without aborting |
| P0 | Analyse a selection or all new files | Create image embeddings and optional caption proposals with provenance |
| P0 | Review and write | Preview proposed changes; accept/edit/reject; create or safely merge sidecars |
| P0 | Search | Return thumbnails for natural-language queries and exact metadata filters |
| P0 | Resume | Continue after application termination without corrupting outputs |
| P1 | Find similar | Rank related photos from a selected or supplied image |
| P1 | Scheduled scan/watch | Detect changes; reconcile periodically when file events are missed |
| P1 | Controlled unattended processing | Automatically write eligible suggestions to application-owned sidecars under an explicit saved policy |
| P2 | Advanced organisation | Optional clusters, near-duplicate review, richer taxonomy and offline OCR |

Initial format support: JPEG/JPG, PNG, TIFF, NEF, RAF and DNG. HEIC/HEIF and additional RAW families are optional until decoder packaging and representative fixtures pass. A file extension alone must not imply successful decoding. RAW acceptance must include real Nikon D800E and Fujifilm X-H2 samples, including compression variants used by the owner. LibRaw's camera list is a starting point, not a substitute for testing files. [S9]

## 5. Functional requirements

### FR-01 — Discovery and file identity

Each configured root has a stable ID, absolute current location, inclusion/exclusion patterns, sidecar policy and availability status. Store paths relative to the root so a library can be remounted elsewhere.

Use size and modification time as a fast change hint; calculate a streaming SHA-256 before first analysis and when content may have changed. Offer a full verification scan for changes that preserve timestamps and size. Keep separate identities for a file instance and its content hash: identical copies may share analysis but retain distinct locations and sidecars.

Skip symbolic links by default; an opt-in mode must detect cycles and root escapes. Ignore application caches, temporary files, sidecars and backups as photo inputs. Support Unicode, spaces, case-sensitive and case-insensitive filesystems. Defer files still being copied until stability checks pass. Recheck the source fingerprint before committing analysis.

An unavailable NAS root must be marked offline, never interpreted as deletion of every photo. Missing paths are tombstoned only after a successful complete scan of an available root. Release 1 does not delete source images.

### FR-02 — Technical metadata and previews

Use ExifTool through a process adapter with structured JSON output. Read capture time, timezone offset if present, camera, lens, exposure, aperture, ISO, focal length, dimensions, orientation, GPS if present, embedded descriptive metadata and recognised sidecars. Preserve raw values when normalisation is ambiguous. Never substitute filesystem modification time for capture time without labelling the fallback.

Prefer a sufficiently large embedded RAW preview; otherwise render using a LibRaw-based adapter. Apply EXIF orientation exactly once, normalise the analysis image to sRGB, and preserve aspect ratio. The model adapter applies its own required final preprocessing. Record preview source, extraction tool version, processing recipe and derivative hash. A preview is not a colour-critical RAW rendering.

Use bounded local caches and separate thumbnail and inference derivatives. Remove unnecessary EXIF/GPS from inference derivatives. Protect against decompression bombs with pixel, memory and elapsed-time limits. A decoding failure becomes an actionable per-file error.

### FR-03 — Embeddings and search

Embed actual image pixels, not merely captions. Keep optional caption-text vectors in a separate named vector collection. A text-only fallback must be visibly labelled as degraded and must never be presented as direct visual search.

Every vector collection has an immutable embedding-space identity: model revision/digest, encoder selection, output dimensions, precision/quantisation, preprocessing version, task configuration and normalisation. Record the compatible query recipe. Query and image roles can use different documented prefixes, but must belong to the same tested retrieval configuration. Reject mixed spaces and non-finite, zero-norm or wrong-length vectors.

Use cosine similarity with appropriately normalised vectors. Apply structured date/camera/root/file-type filters during candidate selection, not only after retrieving a tiny global top-K. Use deterministic tie-breaking and pagination. Similarity is a ranking score, not a probability that a tag is true.

Reindex into a new collection after a model or incompatible processing change, then switch atomically once complete. Keep the previous index usable until replacement succeeds. User metadata corrections must not require regenerating pixel embeddings.

### FR-04 — Descriptive metadata

The caption model produces schema-validated JSON with a short title, one or two factual descriptive sentences, up to 20 concise keywords, optional scene/activity/object categories, and warnings or uncertainty notes. English is the initial output language; preserve existing language alternatives in XMP.

Prefer visible content over speculative context. Do not invent names, exact locations, dates, events or relationships. Explicit user labels and verified EXIF may be used with their source retained. GPS and identifying metadata are not passed into caption prompts by default. Visible text must be marked unverified unless confirmed; a separate OCR adapter can be added later.

Treat text within photos, EXIF and filenames as untrusted data, never instructions. Models have no filesystem write permissions or tool access. Validate lengths, Unicode, enumerations, array limits and unexpected fields. Retry malformed output once with a bounded repair request; then retain the failure for review.

Do not use self-reported model confidence as a calibrated acceptance probability. Default generated descriptions and tags to “proposed”. Optional vocabulary matching uses labelled calibration data and remains separate from free-form generation.

### FR-05 — Review, overrides and provenance

Keep extracted source metadata, generated proposals and effective user-approved values distinct. Record the source of each field, model digest, prompt/schema version, generation timestamp, source image hash, and user decisions. User edits create locked overrides. Rejected tags remain suppressed on regeneration unless the user resets that decision.

The interface must show existing versus proposed title/description/keywords and explain why a change is blocked. Batch acceptance is allowed. Metadata generation can be disabled while visual search remains available; caption runtime failures must not block completed image embeddings.

### FR-06 — XMP export

Write XMP as UTF-8 RDF/XML through a proper serializer. The LLM must never generate the XML document. XMP standards define portable descriptive properties, but sidecar discovery and supported fields differ between receiving applications. [S6, S7]

| Policy | Example for `image.jpg` | Intended use |
|---|---|---|
| Application-owned, recommended default | `image.jpg-about.xmp` | Collision-resistant adjacent metadata; catalogue knows the association |
| Requested short form | `image-about.xmp` | Allowed when the filename stem uniquely identifies a file in that directory |
| Photo-editor compatibility profile | `image.xmp` or `image.jpg.xmp`, as validated per editor | Explicit opt-in after collision and existing-sidecar checks |

`image-about.xmp` is a custom naming convention. Do not claim that Lightroom, darktable, digiKam or another editor will automatically load it. Same-stem RAW+JPEG files must not silently share one sidecar. Refuse ambiguous naming, or require an explicit grouping/export decision. Sidecar ownership is established by an application marker plus matching source binding, not by filename alone.

| Information | XMP representation | Ownership rule |
|---|---|---|
| Title | `dc:title`, language alternatives | Fill only when allowed; preserve user/localised values |
| Description | `dc:description`, language alternatives | Approved description, or clearly marked generated value in app-owned export |
| Keywords | `dc:subject`, unordered string collection | Deduplicate; preserve manually supplied terms |
| Metadata update time | `xmp:MetadataDate` | Actual successful metadata change; unchanged runs do not rewrite it |
| Writer | `xmp:CreatorTool` | Identify this application; preserve prior provenance in history |
| AI provenance | Private versioned namespace | Source filename/hash, analysis ID, model/prompt versions, review state |
| Rating, creator, rights, GPS, capture time | Existing appropriate properties | Preserve; AI never manufactures or changes them |

Assign a stable project-owned namespace URI before implementation; do not misuse Adobe/IPTC properties for private data. Standard descriptions use `rdf:Alt` with `xml:lang="x-default"`; keywords use `rdf:Bag`. [S6] Store vectors in the catalogue, not XMP. Do not copy all EXIF indiscriminately into every sidecar.

Write modes:

1. **Dry run:** show filenames and semantic diffs without changing photo directories.
2. **Create application sidecar:** default export target; fail on unexpected existing content.
3. **Update owned sidecar:** replace only app-owned values; preserve later external edits using a three-way comparison against the last exported snapshot.
4. **Merge editor sidecar:** explicit opt-in; fill missing descriptive values and add approved keywords while preserving unknown namespaces, processing settings and manual fields.

Never remove a keyword unless provenance proves the application owns it and it has not been manually adopted or modified. If ownership cannot be established, preserve it and ask for review. Invalid or encrypted/unparseable XMP is never overwritten. Disallow DTDs and external entities during XML parsing.

For each write: calculate destination fingerprint, save a recoverable previous version if replacing, serialize to an exclusive temporary file in the destination directory, parse-validate, recheck destination and source fingerprints, flush, and replace atomically where supported. Enforce a single catalogue writer and serialize writes per destination. Local locks cannot control unrelated editors: compatibility merges require a quiescent destination, and filesystems without reliable replacement semantics must fail safely or use an explicitly tested strategy. Store a write-intent journal so recovery can reconcile a filesystem commit with a database commit. No cross-system transaction is assumed.

Unchanged runs must leave sidecar bytes and timestamps unchanged. A write failure must preserve the prior sidecar and source image. Read-only roots remain searchable, with pending export and optional explicit mirrored export to another root. Never silently write to the original as an alternative to an unavailable sidecar. ExifTool provides established sidecar workflows, but application policy must constrain destinations. [S8]

## 6. Architecture and language choice

| Option | Advantages for this project | Trade-off | Decision |
|---|---|---|---|
| Java + Quarkus | Familiar stack, strong JSON/HTTP support, maintainable service and CLI | JVM footprint; native image/metadata tooling still needed | Recommended |
| Go | Straightforward service/CLI deployment and concurrency | Image/RAW work still relies on adapters; less benefit from existing Java experience | Good alternative |
| Rust | Resource control and useful native integration | More implementation effort for this owner's existing stack | Choose if Rust learning or a compact desktop binary is a primary goal |

Inference and RAW decoding are likely to dominate processing time; benchmark before choosing a language for assumed speed. Keep the domain model and provider contracts language-neutral.

Components: CLI and local web UI; application services; root scanner; durable job queue; metadata/preview adapters; embedding adapter; caption adapter; metadata policy engine; XMP writer; SQLite catalogue; replaceable vector index.

Use SQLite on a local SSD with a single logical writer and explicit migrations. Do not put the active SQLite database on SMB/NFS. Source photos and adjacent sidecars may remain on a NAS. Start with persisted float32 vectors and an exact-search implementation for correctness; benchmark a maintained embedded ANN backend behind `VectorIndex` before promising 100,000-photo interactive performance. No separate vector database service is required for the first implementation.

At 768 float32 dimensions, one vector uses 3,072 bytes: 100,000 image vectors need 307.2 MB decimal, excluding indexes, row overhead, thumbnails and extra vector collections. Caption-text vectors approximately double raw vector storage if enabled. These are calculated storage estimates, not measured application requirements.

### Logical data model

| Entity | Principal fields |
|---|---|
| Root | ID, location, availability, naming/export policy |
| Asset | ID, content hash, media format, technical metadata |
| FileInstance | ID, asset ID, root ID, relative path, size, mtime, presence state |
| Derivative | Asset ID, recipe/version, hash, local cache path |
| Analysis | Asset ID, provider/model/prompt versions, proposals, warnings, state |
| FieldDecision | File or asset scope, field, value, provenance, lock/rejection state |
| Embedding | Asset/derivative ID, space ID, vector, generation state |
| SidecarState | File instance, destination, last exported snapshot/hash, current status |
| Job | Stage, input fingerprint, attempts, lease, next attempt, error category |
| WriteIntent | Destination, expected old/new hashes, backup reference, commit state |

Metadata overrides default to file-instance scope; sharing them across duplicates is explicit. A RAW/JPEG pair is not an exact duplicate simply because filenames match.

## 7. Provider contracts and scheduling

Define application-owned operations `capabilities`, `embedImages`, `embedQueries`, `describeImage`, `health` and `modelIdentity`. These are internal interfaces, not assertions about Ollama endpoints. Each adapter maps them to a verified transport.

`capabilities` returns supported modalities, dimensions, batching limits, maximum image/input sizes, schema support and model identity. `embedImages` accepts derivative bytes plus a MIME type and recipe ID; it does not send local server paths to a remote worker. Results preserve input ordering or carry explicit request IDs.

Ollama's documented `/api/embed` is the initial text adapter; `/api/chat` with vision input and a JSON schema is the caption adapter. The image-embedding mapping is a Phase 0 deliverable. A fallback local service must perform genuine EmbeddingGemma 2 image encoding; embedding a filename or base64 string as ordinary text is invalid.

Persist stage-specific jobs: discover → extract → preview, then independent embed and describe stages, followed by review/policy and export. Search can become ready before captions finish. States include pending, running, succeeded, retryable failure, blocked, needs review and permanently failed.

Use leases for interrupted jobs and idempotency keys derived from input hash, stage and recipe. Bound I/O, decoder and inference concurrency independently. Default to one inference request at a time and benchmark batching. If GPU memory is constrained, schedule embedding and caption phases separately to avoid repeated model eviction. Respect cancellation between assets and enforce request timeouts. Retry transient errors with bounded exponential backoff; do not retry malformed files indefinitely.

## 8. User interface and CLI

The local web UI contains Library, Photo details, Search, Review queue, Jobs and Settings. Photo details display the preview, technical facts, proposals, user overrides, processing state and sidecar path. Search offers text input, image query, filters and a thumbnail grid. Review includes before/after changes and batch actions.

The headless CLI is first-class. Proposed commands below describe the application to be built, not existing software:

```text
photocat doctor
photocat root add /photos --sidecar-template '{filename}-about.xmp'
photocat scan --all
photocat analyse --pending
photocat search 'swimmers diving into a pool' --limit 30
photocat similar /photos/example.jpg --limit 20
photocat export --approved --dry-run
photocat export --approved
photocat retry --stage embedding
photocat verify --original-hashes
photocat serve --bind 127.0.0.1
```

Provide machine-readable JSON output, useful exit codes, progress totals and summaries of skipped/failed files. `doctor` checks actual image embeddings, query compatibility, caption JSON, decoder availability, permissions, disk space and runtime model identity. Do not auto-download models during an ordinary scan.

## 9. Privacy, deployment and operations

Default to loopback-only services and zero external inference. Allow an explicitly configured private LAN worker as a separate mode; disclose that derivatives travel to that host, require authentication and protected transport, and prohibit silent cloud fallback. Disable unsolicited telemetry, model downloads and update checks in offline mode.

Keep prompts, previews, metadata and thumbnails private. Avoid logging captions, GPS and full personal paths by default. Application APIs must not accept arbitrary filesystem paths outside registered roots. Browser mutations need origin/CSRF protection even on localhost. Run decoder processes with time/memory limits and pass arguments directly rather than through shell interpolation.

Initial deployments: Linux x86-64 workstation/server; macOS ARM64 and Windows x86-64 after platform acceptance. Test GPU inference on the available NVIDIA RTX 4070 Ti Super 16 GB and Apple Silicon where available. These are test targets, not guaranteed throughput or model-fit claims. A NAS hosts images; it need not host the models. An SBC may run the catalogue with a LAN inference worker after measurement; accelerator support must be proven separately.

Back up the catalogue, approved metadata/decisions, configuration, sidecars and write journal using a consistent snapshot. Vector indexes and derivatives are rebuildable. Catalogue loss must not erase approved descriptions already exported; a recovery scan imports app-owned sidecars and rebuilds vectors. Model binaries can be redownloaded if their exact revisions remain available. Include dependency/model license notices and a versioned runtime manifest.

## 10. Acceptance criteria and quality targets

These are proposed release gates and targets, not claims of measured performance.

| ID | Scenario | Pass condition |
|---|---|---|
| AC-01 | Offline end-to-end run | With dependencies preinstalled and outbound network blocked, scan, embed, describe, search and export succeed |
| AC-02 | Original preservation | SHA-256 of every source in the test corpus is identical before and after all workflows |
| AC-03 | Incremental rerun | Unchanged run performs no inference and no sidecar rewrites |
| AC-04 | Sidecar preservation | Unknown namespaces, edit settings, ratings, multilingual fields and manual keywords survive merge fixtures |
| AC-05 | Name collision | `image.NEF` and `image.JPG` never overwrite each other's metadata; short-name conflict is reported |
| AC-06 | Crash recovery | Forced termination during each stage resumes; before/after sidecar replacement is reconciled without invalid XML |
| AC-07 | External edit | An edit made after planning is detected where observable; conflicting merges stop rather than replace user content |
| AC-08 | RAW and orientation | Representative NEF/RAF/DNG fixtures render correctly oriented, including portrait and mirrored EXIF cases |
| AC-09 | Genuine multimodal inference | Dissimilar test images produce finite, distinct vectors; matching descriptions rank appropriate images above distractors |
| AC-10 | Model change | New model vectors remain isolated; the existing search index works until the new collection is promoted |
| AC-11 | NAS unavailable | Root goes offline without mass deletions or loss of catalogue/decisions |
| AC-12 | Invalid model output | Malformed JSON is retried once, then quarantined; no invalid metadata is written |
| AC-13 | User overrides | Accepted corrections and rejected tags survive rescans and regeneration |
| AC-14 | Rebuild | Fresh catalogue can recover exported approved descriptive metadata and source associations |
| AC-15 | Compatibility | At least one chosen photo editor recognises the tested naming profile and reads title/description/keywords; report results per format |

Build a representative evaluation collection of approximately 500 photos across people, swimming, pets, landscapes, low light, screenshots, orientation cases and near duplicates. Create at least 50 human-labelled queries. Initial quality targets: a relevant result in the top 10 for at least 85% of answerable queries; at least 90% of sampled generated keyword assignments visibly supported; at least 80% of descriptions accepted unchanged or with only minor edits. Include “no suitable image” queries and use them to evaluate misleading matches; do not turn every search into a forced match.

Measure warm-search p95 with 100,000 images; target under two seconds on the selected reference machine, reporting query embedding and index time separately. Measure indexing photos/minute, caption latency, RAW extraction latency, GPU/CPU memory and cache growth on the actual hardware before committing to throughput. A speed target cannot override preservation requirements. Search should remain responsive during indexing through separate bounded queues and read access.

## 11. Delivery plan

| Phase | Deliverable | Exit gate |
|---|---|---|
| 0 — Feasibility | Runtime matrix; genuine local image/query embeddings; caption comparison; NEF/RAF previews; one XMP round trip | Exact versions and request formats recorded; no unresolved blocker to local operation |
| 1 — Catalogue | CLI, roots, SQLite, extraction, previews, hashes, durable stages | Incremental/crash/offline-root tests pass |
| 2 — Visual search | Pixel embeddings, filters, text and similar-image search | Retrieval evaluation and model-space isolation pass |
| 3 — Metadata | Caption proposals, user overrides, app-owned XMP, dry-run and journal | Original/sidecar preservation and review tests pass |
| 4 — Usability | Local UI, compatibility profile, unattended policy, packaging | Target-platform and photo-editor acceptance pass |
| 5 — Optional improvements | ANN tuning, clustering, OCR, additional formats | Each feature evaluated independently |

First usable milestone is a CLI that scans a folder, searches actual image embeddings and writes reviewed app-owned XMP for selected files. Do not delay this for a polished desktop shell or multi-user infrastructure.

## 12. Decisions to confirm during implementation

Defaults are sufficient to start Phase 0; these are not blockers to the PRD.

- Preferred receiving photo editor determines the compatibility naming/merge profile.
- Estimated collection size and NAS filesystem determine scale and write testing.
- Primary operating system and inference host determine the first certified runtime path.
- Whether unattended generated sidecars should contain unreviewed descriptions or only approved fields; default is approved export, with explicit opt-in for unreviewed app-owned export.
- Whether short `image-about.xmp` naming outweighs the safer `image.jpg-about.xmp` convention.
- Exact caption model, supported runtime version and embedded vector backend are selected through Phase 0/2 measurements.

## 13. Source register

Public documentation checked 11 October 2026. Runtime support changes quickly; pin and test the chosen build. Requirements and architecture above are proposed product decisions unless explicitly attributed.

- **S1 — Google model card:** https://ai.google.dev/gemma/docs/embeddinggemma/model_card_2 — model purpose, modalities and dimensions.
- **S2 — Ollama model listing:** https://ollama.com/library/embeddinggemma-2 and https://ollama.com/library/embeddinggemma-2:440m — published model availability; not a cross-platform integration certification.
- **S3 — Ollama embedding API:** https://docs.ollama.com/api/embed — published embedding endpoint and text input schema.
- **S4 — Google model repository:** https://huggingface.co/google/embeddinggemma-2 — model files and implementation instructions to pin during integration.
- **S5 — Ollama structured outputs:** https://docs.ollama.com/capabilities/structured-outputs — structured generation and vision example.
- **S6 — Adobe Dublin Core XMP:** https://developer.adobe.com/xmp/docs/xmp-namespaces/dc/ — descriptive metadata properties.
- **S7 — Adobe XMP:** https://developer.adobe.com/xmp/docs/ and https://developer.adobe.com/xmp/docs/xmp-namespaces/xmp/ — XMP and basic metadata properties.
- **S8 — ExifTool sidecars:** https://exiftool.org/metafiles.html — sidecar creation/copying workflows.
- **S9 — LibRaw camera coverage:** https://www.libraw.org/supported-cameras — camera support dependent on build/features.
