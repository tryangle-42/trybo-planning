# Data/File Formats — Content Extraction & Agent Context

## Context

When a user uploads a file in the chat, the **File Attachment for Outbound Communications** component stores the binary in a private Supabase Storage bucket and persists file metadata (doc_id, filename, mime_type, storage_path) in **chat memory** under the chat_id. That component explicitly does **not** parse or extract content — it treats all file types identically as binaries.

This component picks up where the file attachment component stops. It retrieves the uploaded file from storage, extracts structured content based on the file type, and writes a **compressed summary** into chat memory. Chat memory stays lean — only summaries, never full text or full data.

The extraction strategy is **consistent across all file types**:

- **Chat memory:** `content_summary` at **10% of the original character count** (e.g., 1,000-char doc gets ~100-char summary, 50,000-char doc gets ~5,000-char summary). Enough for the agent to understand what the file contains and decide whether it needs the full content.
- **skill_data_store:** full extracted content (keyed by doc_id) — full text for documents, full transcript for audio, full parsed tabular data for CSV/XLS/XLSX. The agent fetches this on demand when it needs exact content to answer the user's question.

```
ALL file types — same pattern:
  Chat memory:      10% summary (always lean)
  skill_data_store: full extracted content (on-demand fetch by doc_id)
  Storage bucket:   original binary file (managed by File Attachment Outbound)

Agent flow (same for all types):
  1. Loads chat memory → sees 10% summary → understands what the file is
  2. Needs exact content → fetches from skill_data_store by doc_id
  3. Reasons over full content → answers user
```

The original file binary always remains in the storage bucket. Chat memory is lean — only summaries, never full text or full data. One consistent pattern, no special cases per file type.

### Critical: Parallel Extraction — User Never Blocked

The file upload pipeline has two steps — binary storage (File Attachment Outbound) and content extraction (this component). These must both complete before the **agentic orchestration** can start. But the user is **never blocked** — not at upload time, not at send time, not ever.

**The design:** extraction runs in the background after upload. The user can type and send their message at any time. The **agentic orchestration** (not the user) waits for extraction to complete before starting.

```
WRONG (race condition — no gate):
  Upload → binary stored → user sends → agentic orchestration starts
                                              ↓
                   extraction still running... agent sees NO content

WRONG (block the user):
  Upload → binary stored → extraction runs → user waits... → user can type → sends

RIGHT (send blocked only during file upload, not during extraction):
  Upload starts → send blocked during file upload
    → File Attachment Outbound: binary stored + metadata in chat memory
  Upload completes → send unblocked → chip shows

  Extraction starts in background → user types → user sends → message saved
                 ↓                                                ↓
         extraction running                             backend: extraction done?
                 ↓                                      YES → fire orchestration
         extraction completes                           NO  → wait for extraction
                 ↓                                            → content written to chat memory
         content in chat memory                               → then fire orchestration
```

**Why this works:**
- **Send is only blocked during file upload** (File Attachment Outbound — binary storage + metadata to chat memory). This is fast for all file types.
- **Send is NOT blocked during extraction.** Once the file upload completes, the user can type and send immediately.
- The extraction delay is absorbed into the **agent response time** — which the user is already expecting to wait for.
- For **documents and tabular** (< 1 second extraction): extraction finishes before the user even finishes typing. By the time the message arrives at the backend, chat memory already has everything. Zero additional delay to agent response.
- For **audio** (10-30 seconds): extraction overlaps with the user's typing time. Even if the user sends before transcription finishes, the additional wait looks like normal agent "thinking" time.
- **The guarantee is preserved:** the agentic orchestration never starts until extracted content is in chat memory. The agent always sees everything.

**What this component does NOT need to do on the frontend:**
- No "processing" chip state. No spinner. No disabled send button. No frontend polling for extraction status.
- The chip shows as soon as the file upload (File Attachment Outbound) completes. That's it.
- All extraction coordination is backend-only — the message send endpoint checks extraction status before starting the orchestration.

### Goals

1. **Format-aware content extraction** — each file type is parsed with a specialised strategy (text extraction, tabular parsing, audio transcription)
2. **Lean chat memory — 10% summary ratio** — chat memory gets only a compressed summary (10% of original character count), never full text or full data. Chat memory stays lean even with many files.
3. **Full content accessible on demand** — full extracted content (text, transcript, or parsed tabular data) stored in skill_data_store, agent fetches by doc_id when it needs exact content. Same pattern for all file types.
4. **Chat-scoped lifecycle** — summaries in chat memory follow the chat lifecycle. Full extracted text in skill_data_store is cleaned up when the file is deleted.

### Supported File Types

| Category | Formats | Extraction Strategy |
|----------|---------|---------------------|
| **Text documents** | PDF, DOCX, TXT | Text extraction → plain text |
| **Tabular data** | CSV, XLS, XLSX | Parse → structured rows/columns + text summary |
| **Audio** | MP3, WAV, M4A, OGG | Transcription → plain text transcript |

---

## Component Responsibility

```
┌──────────────────────────────────────────────────────────────────┐
│     UPLOAD PIPELINE (parallel extraction, gate at send)          │
│                                                                  │
│  Step 1: FILE ATTACHMENT OUTBOUND (immediate)                    │
│     Store binary in bucket                                       │
│     Write file metadata to chat memory under chat_id             │
│     → Upload returns to frontend                                 │
│     → File chip shows in UI                                      │
│     → User can start typing immediately                          │
│                                                                  │
│  Step 2: THIS COMPONENT (runs in parallel with user typing)      │
│     Retrieve binary from storage                                 │
│     Detect file type → select parser → extract content           │
│                                                                  │
│     All file types — same pattern:                               │
│       Extract full content (text / transcript / parsed data)     │
│       Store full content in skill_data_store (by doc_id)         │
│       Generate 10% summary → write to chat memory                │
│       Agent fetches full content from skill_data_store on demand │
│                                                                  │
│  ORCHESTRATION GATE (backend-only, user never blocked):          │
│     User sends message → message saved immediately               │
│     Backend checks: is extraction complete?                      │
│     YES → fire agentic orchestration (agent has everything)      │
│     NO  → wait for extraction → content in chat memory           │
│           → then fire agentic orchestration                      │
│     User is NOT blocked — delay looks like agent "thinking"      │
│                                                                  │
└──────────────────────────────┬───────────────────────────────────┘
                               │  (agent loads chat memory → sees everything)
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│     NOT THIS COMPONENT'S RESPONSIBILITY                          │
│                                                                  │
│  - Storing the original binary file                              │
│    → handled by File Attachment Outbound component               │
│                                                                  │
│  - Deciding what the agent does with the content                 │
│    → agent's own judgment based on user request                  │
│                                                                  │
│  - Delivering files to channels (WhatsApp, email, etc.)          │
│    → each channel handles delivery on its own                    │
│                                                                  │
│  - Embedding or vector search over document content              │
│    → future enhancement, not in scope                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Should Work

### Content Extraction Flow (Per-File, Immediate, Parallel)

Extraction is triggered **per file, immediately** when that individual file's upload completes. It does NOT batch. It does NOT wait for all files to be uploaded. It does NOT wait for the user to send a message. Each file's extraction runs independently and in parallel with other extractions and with the user typing.

```
User uploads file A (e.g. sales.csv)
  -> File Attachment Outbound completes for A -> chip shows
  -> Extraction A starts IMMEDIATELY in background (< 1 sec for CSV)

User uploads file B (e.g. contract.docx)      <- while extraction A running
  -> File Attachment Outbound completes for B -> chip shows
  -> Extraction B starts IMMEDIATELY in background (< 1 sec for DOCX)

User uploads file C (e.g. meeting.m4a)         <- extraction A already done
  -> File Attachment Outbound completes for C -> chip shows
  -> Extraction C starts IMMEDIATELY in background (10-30 sec for audio)

All three extractions run independently and in parallel.
User types their message while extractions are running.

User sends message -> message saved immediately (user NOT blocked)
  -> Backend checks: are extractions for ALL attached files complete?
  -> YES -> fire agentic orchestration -> agent sees all content
  -> NO (e.g. C still transcribing) -> orchestration waits
       -> C finishes -> content written to chat memory
       -> fire orchestration -> agent sees everything
  -> User perceives wait as normal agent "thinking" time
```

**Per-file extraction detail:**

```
Each file, independently, as soon as its upload completes:

  -> Retrieve file binary from storage
  -> Route to appropriate parser based on mime_type / extension:

    PDF / DOCX / TXT (< 1 second):
      -> Extract plain text (PyMuPDF, python-docx, plain decode)
      -> Preserve structure: headings, paragraphs, lists where possible
      -> Capture metadata: title, author, page count

    CSV (< 1 second):
      -> Parse with column headers and row data
      -> Generate data summary: column names, row count, data types, sample rows
      -> Full content as structured text (headers + rows)

    XLS / XLSX (< 1 second):
      -> Parse each sheet (or active sheet) into rows/columns
      -> Same data summary approach as CSV
      -> Handle multiple sheets: list sheet names, parse primary sheet

    MP3 / WAV / M4A / OGG (10-30 seconds):
      -> Transcribe audio using speech-to-text (ElevenLabs STT API)
      -> Return plain text transcript with timestamps (if available)
      -> Capture metadata: duration, language detected

  -> Store full extracted content in skill_data_store (keyed by doc_id)
  -> Generate 10% summary -> write summary to chat memory under same chat_id
  -> This file's extraction is done -- independent of other files
```

### How the Agent Sees Extracted Content

```
Agent loads chat memory for a conversation
  -> File metadata is there (written by File Attachment Outbound)
  -> 10% content summary is there (written by this component)
  -> Because the orchestration gate ensures all extractions are complete
      before the agentic orchestration starts, summaries are ALWAYS present
  -> For each file, the agent sees in chat memory:
      - doc_id, filename, mime_type, storage_path  (from File Attachment Outbound)
      - content_type: "document" | "tabular" | "audio_transcript"
      - content_summary: 10% compressed summary of the file content
      - extraction_metadata: {row_count, column_names, duration, page_count, etc.}
  -> No "pending" state — orchestration gate guarantees completion

When the agent needs exact content:
  -> Agent calls skill_data_store.fetch(doc_id)
  -> Gets full extracted text / full transcript / full parsed tabular data
  -> Reasons over exact content -> answers user's question
  -> Same fetch pattern for all file types
```

### Example Flows

**CSV Upload (zero perceived delay):**
```
User uploads sales_q1.csv (2,500 chars of tabular data)
  -> Upload completes -> chip shows -> extraction starts in background
  -> CSV parsed in < 1 second: 1,247 rows, 5 columns
  -> Full parsed data stored in skill_data_store
  -> 10% summary (~250 chars) written to chat memory:
       "sales_q1.csv: 1,247 rows, 5 columns (date, product, region,
        revenue, units). Date range: 2025-01-01 to 2025-03-31.
        Sample: [first 3 rows]"
  -> User types "summarise Q1 sales by region" -> sends immediately
  -> Orchestration starts -> agent reads summary from chat memory
  -> Agent decides: needs exact data -> fetches from skill_data_store
  -> Reasons over full tabular data -> generates regional summary
```

**DOCX Upload (zero perceived delay):**
```
User uploads contract_draft.docx (42,000 chars)
  -> Upload completes -> chip shows -> extraction starts in background
  -> Text extracted in < 1 second: 4,200 words, 12 pages
  -> Full text stored in skill_data_store
  -> 10% summary (~4,200 chars) written to chat memory:
       compressed summary covering key sections, clauses, parties involved
  -> User types "what are the payment terms?" -> sends immediately
  -> Orchestration starts -> agent reads summary from chat memory
  -> Agent decides: needs exact payment clause -> fetches from skill_data_store
  -> Finds payment terms in full text -> answers user
```

**Audio Upload (delay absorbed into agent response time):**
```
User uploads meeting_notes.m4a (23 min recording)
  -> Upload completes -> chip shows -> user starts typing immediately
  -> ElevenLabs transcription runs in background (~20 seconds)
  -> Transcript: 34,000 chars
  -> Full transcript stored in skill_data_store
  -> 10% summary (~3,400 chars) written to chat memory:
       compressed summary of discussion topics, decisions, participants
  -> User types "what were the action items?" -> sends immediately
  -> Backend: extraction done? YES (user typed for 15+ seconds)
  -> Orchestration starts -> agent reads summary from chat memory
  -> Agent decides: needs exact transcript -> fetches from skill_data_store
  -> Scans full transcript -> extracts action items
```

---

## Design Phases

### Phase 1: Extend Parser Registry (CSV + XLS/XLSX)

Add CSV and Excel parsers to the existing `document_parsers/` registry. The parser interface (`DocumentParser` protocol) is already pluggable — implement `supports()` and `parse()`, register in `registry.py`.

**CSV Parser:**
- Parse using Python's `csv` module (stdlib — no new dependency)
- Detect headers from first row
- Extract all rows as structured data
- Generate data summary: column names, row count, data types inferred from sample, first 5 rows as sample
- Output plain_text as formatted table + summary block

**XLS/XLSX Parser:**
- Parse using `openpyxl` for XLSX (already available in many Python stacks) and `xlrd` for legacy XLS
- Handle multiple sheets: list sheet names, parse active/first sheet by default
- Same output format as CSV: headers, rows, summary
- Include sheet metadata in `ParsedDocument.metadata`

**Update `document_service.py`:**
- Expand `ALLOWED_EXTENSIONS` to include `.csv`, `.xls`, `.xlsx`
- Register new parsers in `get_parser_registry()`

---

### Phase 2: Audio Transcription

Add audio transcription capability for MP3, WAV, M4A, OGG files.

**Audio Parser:**
- Use ElevenLabs STT (Scribe) API for transcription — already integrated in the system (`asr_elevenlabs.py` for real-time call transcription). Adapt for file-based audio.
- Accept audio bytes, send to ElevenLabs Scribe API, receive transcript text
- Capture metadata: duration (from audio headers or ElevenLabs response), detected language
- Handle large audio files: if file exceeds API limit, split into chunks and concatenate transcripts
- Output plain_text as full transcript, page_count represents approximate "pages" of transcript

**Fallback:** If ElevenLabs STT API is unavailable or user hasn't configured ElevenLabs, return a clear error indicating transcription is not available rather than silently failing.

**Update `document_service.py`:**
- Expand `ALLOWED_EXTENSIONS` to include `.mp3`, `.wav`, `.m4a`, `.ogg`
- Register audio parser in `get_parser_registry()`

---

### Phase 3: 10% Summary Generation + skill_data_store Storage

For each file, this phase produces two outputs:
1. **Full extracted content** stored in `skill_data_store` (keyed by doc_id) — agent fetches on demand
2. **10% summary** written to chat memory — agent always sees this in chat context

**10% Summary Rule:**
The summary character count = **10% of the original content's character count.** The LLM is instructed to compress the content into this target length.

| Original Size | Summary Target | Example |
|---|---|---|
| 500 chars | ~50 chars | One sentence |
| 5,000 chars | ~500 chars | Short paragraph |
| 50,000 chars | ~5,000 chars | Detailed multi-paragraph summary |

**Summary Strategy by Type:**
- **Documents** (PDF, DOCX, TXT): LLM-generated summary at 10% of the extracted text length. The LLM is given the full text and instructed: "Summarize in approximately N characters."
- **Tabular** (CSV, XLS, XLSX): Deterministic summary (no LLM) — column names, data types, row count, and as many sample rows as fit within the 10% budget. Template: "{filename}: {row_count} rows, {col_count} columns ({column_names}). Types: {types}. Sample: [rows that fit in budget]"
- **Audio** (MP3, WAV, M4A, OGG): LLM-generated summary at 10% of the transcript length. Includes duration and key topics.

**Storage — two destinations:**
- **skill_data_store** (keyed by doc_id): full extracted content — full text, full transcript, or full parsed tabular data. Agent fetches this when it needs exact content.
- **Chat memory** (under chat_id): 10% summary + extraction_metadata only. Always lean.

---

### Phase 4: Background Extraction with Orchestration Gate

Wire this component to start extraction immediately after File Attachment Outbound completes, running in the background. The user is never blocked. The agentic orchestration waits for extraction to complete before starting.

**Upload Flow:**
- File Attachment Outbound completes (binary stored, metadata in chat memory) → upload returns to frontend
- Backend immediately fires this component's extraction in the background
- Frontend shows file chip → user types and sends at will — **no blocking**
- For documents/tabular: extraction finishes in < 1 second (before user finishes typing)
- For audio: transcription runs 10-30 seconds, overlapping with user's typing time

**Orchestration Gate (backend-only):**
- User sends message → message is saved immediately (user not blocked)
- Backend checks: are all attached files' extractions complete?
- If YES (common case): fire agentic orchestration — agent has everything
- If NO: orchestration waits for extraction to finish → summary written to chat memory, full content in skill_data_store → then fire orchestration
- The agentic orchestration **never starts** without the summary in chat memory and full content in skill_data_store
- User perceives the wait as normal agent "thinking" time — no visible difference

**No Frontend Work for This Component:**
- No "processing" chip state, no spinner, no disabled send button
- The file chip appears as soon as File Attachment Outbound completes — that's all the frontend needs
- All extraction coordination is backend-only

**Extraction Failure Handling:**
- If extraction fails (corrupt file, ElevenLabs unavailable): mark `extraction_status: "failed"` with a reason
- Upload still succeeds — the file is stored, the agent sees the file metadata
- Agent sees a `content_summary` indicating extraction failed ("File uploaded but content could not be extracted: {reason}")
- User can still send — the agent can reference the file by name/URL even if content isn't extracted

**Two-Destination Storage:**
- This component writes to TWO places:
  ```
  skill_data_store (keyed by doc_id):
    - Full extracted content (text / transcript / parsed tabular data)
    - Agent fetches on demand when it needs exact content

  Chat memory for chat_id (alongside file metadata):
    File Attachment Outbound wrote:
      - doc_id, filename, mime_type, storage_path

    This component writes (keyed by same doc_id):
      - content_type: "document" | "tabular" | "audio_transcript"
      - content_summary: 10% compressed summary
      - extraction_metadata: {row_count, column_names, duration, page_count, etc.}
      - extraction_status: "complete" | "failed"
  ```
- Chat memory stays lean — only summaries and metadata, never full content
- Agent loads chat memory → sees summaries → fetches full content from skill_data_store when needed
- No changes to `ExecutionContext` or `contracts.py`

**Chat Memory Gets ONLY the 10% Summary — Never Full Content.**

Chat memory is lean. The only content this component writes to chat memory is the `content_summary` (10% of original size) and `extraction_metadata`. Full extracted content lives in `skill_data_store`, fetched by the agent on demand via `doc_id`.

---

### Phase 5: Lifecycle Alignment

This component writes to two places: chat memory (10% summary) and skill_data_store (full content). Both must be cleaned up on deletion. Pre-send deletion requires special attention because extraction runs in the background and may have already completed.

**Chat Deletion:**
- When the chat is deleted, chat memory is cleaned up — summary goes with it automatically
- skill_data_store records for all doc_ids in that chat must also be deleted
- The deletion cascade: find all doc_ids linked to this chat → delete from skill_data_store → delete from chat memory → proceed with chat deletion

**Pre-Send Removal (cross button) — critical case:**

Because extraction starts immediately after upload and runs in the background, there are three possible states when the user presses the cross button:

```
Case 1: Extraction not started yet (unlikely — extraction fires immediately)
  -> Cancel extraction if possible
  -> Delete file metadata from chat memory
  -> Done — nothing in skill_data_store to clean up

Case 2: Extraction in progress (possible for audio files)
  -> Cancel the running extraction
  -> Delete file metadata + summary from chat memory
  -> If extraction partially wrote to skill_data_store, delete that too
  -> Done

Case 3: Extraction already completed (common — docs/CSV finish in < 1 second)
  -> Delete file metadata + summary from chat memory for this doc_id
  -> Delete full extracted content from skill_data_store for this doc_id
  -> Done — as if the file was never uploaded
```

All three cases must result in the same outcome: **no trace of the file in chat memory** — not the metadata, not the extracted content, not the summary. The agent must never see a deleted file's content.

**Implementation:** The pre-send removal function (triggered by the cross button) must:
1. Cancel any in-progress extraction for this doc_id
2. Delete summary + metadata from chat memory for this doc_id
3. Delete full extracted content from skill_data_store for this doc_id
4. Delete the binary from the storage bucket (File Attachment Outbound's responsibility)

**No Orphans:**
- Extracted content is keyed by doc_id within chat memory
- If the file metadata record is gone, the extraction record must also be gone
- The deletion function handles both in a single operation
- Defensive check: if an extraction completes for a doc_id whose metadata has already been deleted (race condition during cancellation), the extraction should detect this and discard its results rather than writing to chat memory

---

## What Exists Today vs. What's New

| Capability | Exists Today | What's New |
|---|---|---|
| PDF text extraction | Yes — `pdf_parser.py` (PyMuPDF) | No change needed |
| DOCX text extraction | Yes — `docx_parser.py` (python-docx) | No change needed |
| TXT text extraction | Yes — `txt_parser.py` (plain decode) | No change needed |
| CSV parsing | No | New parser: `csv_parser.py` |
| XLS/XLSX parsing | No | New parser: `xlsx_parser.py` |
| Audio transcription | Partial — `asr_elevenlabs.py` exists for real-time streaming | New parser: `audio_parser.py` — adapts ElevenLabs Scribe for file-based audio |
| Content summary generation | No | New: summary layer (LLM for docs/audio, template for tabular) |
| Parser registry | Yes — `registry.py` (PDF, DOCX, TXT) | Extend with CSV, XLSX, Audio parsers |
| `ALLOWED_EXTENSIONS` | `{.pdf, .docx, .txt}` | Expand to include `.csv, .xls, .xlsx, .mp3, .wav, .m4a, .ogg` |
| `document_service.py` | Yes — upload, parse, store in skill_data | Extend to support new types + summaries |
| Two-destination storage | No — documents stored only in skill_data_store | New: 10% summary to chat memory + full content to skill_data_store. Agent sees summary in chat context, fetches full content on demand. |
| Lifecycle cleanup | Partial — `delete_document()` exists | Must clean both chat memory (summary) and skill_data_store (full content) on deletion. Pre-send removal handles 3 extraction states (not started, in progress, completed). |

---

## Implementation Priority & Time Estimates

All implementation is done using Claude. Claude writes all code and unit tests. Human time is for device testing, verification, and bug triage. This component is **backend-only** — no frontend changes needed.

**Working day = 8 hours**

### Phase 1: CSV + XLS/XLSX Parsers
*Dependencies: None (can start immediately — parser registry already exists)*

| Task | Hours |
|------|-------|
| Implement `csv_parser.py` — parse headers, rows, data summary, encoding detection (`chardet`) | 1 |
| Implement `xlsx_parser.py` — openpyxl for XLSX, xlrd for XLS, multi-sheet handling, sheet metadata | 1.5 |
| Add `openpyxl` and `xlrd` to dependencies | 0.25 |
| Register new parsers in `registry.py`, expand `ALLOWED_EXTENSIONS` | 0.25 |
| Unit tests for CSV (headers, empty, large 10K+ rows, non-UTF-8, malformed) | 0.5 |
| Unit tests for XLSX (single sheet, multi-sheet, legacy XLS, empty sheets) | 0.5 |
| **Phase 1 Total** | **4** |

### Phase 2: Audio Transcription (ElevenLabs STT)
*Dependencies: ElevenLabs API key (already configured — `asr_elevenlabs.py` exists)*

| Task | Hours |
|------|-------|
| Implement `audio_parser.py` — ElevenLabs Scribe API integration for file-based audio | 1 |
| Handle large audio files (chunking if exceeds API limit) | 0.5 |
| Audio metadata extraction (duration, format detection, language) | 0.25 |
| Register audio parser in `registry.py`, expand `ALLOWED_EXTENSIONS` | 0.25 |
| Error handling: API unavailable, invalid audio, timeout | 0.25 |
| Unit tests (mock ElevenLabs API, various audio types, chunking, fallback) | 0.5 |
| **Phase 2 Total** | **2.75** |

### Phase 3: 10% Summary Generation + Two-Destination Storage
*Dependencies: P1, P2 (parsers must exist)*

| Task | Hours |
|------|-------|
| Implement 10% summary for documents/audio (LLM-based, target = 10% of char count) | 0.5 |
| Implement 10% summary for tabular (deterministic template — columns, types, row count, sample rows) | 0.25 |
| Two-destination storage: full content to skill_data_store + 10% summary to chat memory | 0.5 |
| Unit tests for summary generation (all file types, various sizes) | 0.5 |
| Unit tests for two-destination storage (verify both stores written correctly) | 0.25 |
| **Phase 3 Total** | **2** |

### Phase 4: Background Extraction + Orchestration Gate
*Dependencies: P1, P2, P3, File Attachment Outbound (upload flow must exist)*

| Task | Hours |
|------|-------|
| Fire extraction in background immediately after each file's upload completes (per-file, parallel) | 0.5 |
| Orchestration gate: message send endpoint checks all extractions complete before starting orchestration | 0.5 |
| Handle orchestration wait (extraction not done when user sends — wait then fire) | 0.5 |
| Extraction failure handling (mark failed, agent sees failure reason in summary) | 0.25 |
| Unit tests for orchestration gate (done before send, pending at send, multiple files, failure case) | 0.5 |
| Integration test: upload + send + verify agent sees summary + fetches full content | 0.5 |
| **Phase 4 Total** | **2.75** |

### Phase 5: Lifecycle Alignment (Pre-Send Deletion + Chat Deletion)
*Dependencies: P4*

| Task | Hours |
|------|-------|
| Pre-send deletion: handle 3 extraction states (not started, in progress, completed) | 0.5 |
| Clean both chat memory (summary) and skill_data_store (full content) on removal | 0.25 |
| Chat deletion cascade: find all doc_ids → delete from skill_data_store → delete from chat memory | 0.25 |
| Race condition guard: extraction completes for already-deleted doc_id → discard results | 0.25 |
| Unit tests for all deletion scenarios | 0.5 |
| **Phase 5 Total** | **1.75** |

### End-to-End Device Testing (iOS + Android)
*After all phases complete*

| Task | Hours |
|------|-------|
| Android build + smoke test | 0.5 |
| Android E2E: documents (PDF, DOCX, TXT) — upload → send → agent uses summary → fetches full content | 1 |
| Android E2E: tabular (CSV, XLSX) — upload → send → agent summarises → fetches full data | 1 |
| Android E2E: audio (MP3, M4A) — upload → send → verify orchestration waits → agent uses transcript | 1 |
| iOS build + smoke test | 0.5 |
| iOS E2E: same test matrix as Android | 1.5 |
| Edge cases: large files, corrupt files, mixed types, pre-send removal (all 3 states), chat deletion, non-UTF-8, send before extraction completes | 1 |
| Bug fixes from testing (Claude fixes, human re-verifies) | 1.5 |
| **E2E Testing Total** | **7** |

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Reuse existing parser registry pattern** | `DocumentParser` protocol + `ParserRegistry` is already proven. New parsers just implement `supports()` + `parse()` and register. No new patterns needed. |
| **ElevenLabs STT (Scribe) for audio transcription** | ElevenLabs Scribe is already integrated in the system (`asr_elevenlabs.py`) for real-time call transcription. Reuse the same provider for file-based audio transcription. No new provider dependency. |
| **Deterministic summaries for tabular data** | CSV/Excel summaries don't need LLM — column names, row counts, and sample data are sufficient and faster/cheaper. LLM used only for docs and audio where semantic understanding is needed. |
| **Background extraction, orchestration gate — user never blocked** | Extraction runs in background after file upload completes. User is never blocked — not at upload, not at send. The agentic orchestration waits for extraction to complete before starting. The user perceives any extraction delay as normal agent thinking time. For docs/tabular (< 1 second), extraction is done before the user finishes typing. For audio (10-30 seconds), transcription overlaps with typing time. |
| **10% summary ratio — chat memory always lean** | Chat memory gets only a 10% compressed summary, never full text or full data. Summary size scales with document size (500-char doc gets ~50-char summary, 50,000-char doc gets ~5,000-char summary). Chat memory stays lean even with many files. |
| **Two-destination storage: chat memory + skill_data_store** | 10% summary goes to chat memory (agent always sees it). Full extracted content goes to skill_data_store (agent fetches on demand by doc_id). Same pattern for all file types — consistent, no special cases. |
| **No frontend work for this component** | The user is never blocked for extraction. No "processing" chip state, no spinner, no disabled send button. The file chip appears when File Attachment Outbound completes. All extraction coordination is backend-only — the message send endpoint checks extraction status before starting orchestration. This means zero frontend changes for Data/File Formats. |
| **Extraction content keyed by doc_id** | Same identifier across all stores. One doc_id = file metadata in chat memory + summary in chat memory + full content in skill_data_store. Clean 1:1 mapping. Deletion by doc_id cleans everything. |
| **No vector embeddings in this phase** | Semantic search over documents is a valuable future enhancement but not needed for the core use case (agent reads extracted content directly). Avoids premature complexity. |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Large CSV/XLSX files (100K+ rows) | Extraction is slow, context is too large for agent | Cap at configurable row limit (default 10,000). Include summary + sample rows in context, not all data. |
| Audio transcription latency (30+ seconds for long recordings) | Agent response is delayed | User is never blocked — they send immediately. Transcription runs in background, overlaps with typing time. If user sends before transcription finishes, the extra wait is absorbed into agent response time. For very long audio (>1 min), the agent response delay could be noticeable — but this is the expected trade-off for transcription quality. |
| ElevenLabs STT API cost for frequent audio uploads | Unexpected API spend | Track transcription minutes per user. Optional: limit per chat or per day. |
| Malformed/corrupt files | Parser crashes | Each parser wraps extraction in try/except. Return `ParsedDocument` with empty text + warning rather than failing the upload entirely. |
| Full extracted content too large for agent context | Agent can't load everything from skill_data_store | Chat memory only has the 10% summary — always fits. Full content in skill_data_store is fetched on demand. If the full content itself is too large for the LLM context when fetched, the agent can request specific sections or apply its own filtering. |
| File Attachment Outbound component not ready yet | Can't wire integration (Phase 4) | Phases 1-3 are independent — build parsers and summaries first. Phase 4 only needs File Attachment Outbound P1-P2 (binary storage + chat memory metadata). Agent awareness (their P3) is no longer a dependency since chat memory is the integration point. |
| Encoding issues in CSV (non-UTF-8) | Garbled text or parser failure | Detect encoding using `chardet` before parsing. Fall back to latin-1 if detection fails. |

---

## Relationship to Existing Code

```
Existing (document_parsers/)          This Component Adds
─────────────────────────             ──────────────────────

base.py (ParsedDocument protocol)     No change — reused as-is
registry.py (ParserRegistry)          Register 3 new parsers
pdf_parser.py                         No change
docx_parser.py                        No change
txt_parser.py                         No change
                                      + csv_parser.py (NEW)
                                      + xlsx_parser.py (NEW)
                                      + audio_parser.py (NEW)

document_service.py                   Expand ALLOWED_EXTENSIONS
                                      + 10% summary generation
                                      + two-destination storage
                                      + pre-send deletion (3 states)

skill_data_store                      Full extracted content stored here
                                      (keyed by doc_id). Agent fetches
                                      on demand. Already exists — reused.

chat memory (under chat_id)           File Attachment Outbound writes
                                      file metadata here. This component
                                      writes 10% summary + metadata only.
                                      No full content in chat memory.
                                      No changes to contracts.py.
```

---

## Total Effort

| Phase | Hours | Days (8h/day) |
|-------|-------|---------------|
| P1 — CSV + XLS/XLSX parsers | 4 | 0.5 |
| P2 — Audio transcription (ElevenLabs STT) | 2.75 | 0.35 |
| P3 — 10% summary generation + two-destination storage | 2 | 0.25 |
| P4 — Background extraction + orchestration gate | 2.75 | 0.35 |
| P5 — Lifecycle alignment (pre-send deletion + chat deletion) | 1.75 | 0.2 |
| End-to-end device testing (Android + iOS) | 7 | ~1 |
| **Total** | **20.25 hours** | **~2.5 days** |

**Recommended order:** P1 + P2 in parallel (day 1 morning) → P3 (day 1 afternoon) → P4 (day 1 afternoon, once File Attachment Outbound upload flow exists) → P5 (day 2 morning) → E2E device testing (day 2 afternoon + day 3 morning)

Note: P1 and P2 can be worked on in parallel since CSV/XLSX and audio parsers are independent. P3 depends on parsers existing. P4 depends on File Attachment Outbound upload flow being ready. P5 depends on P4.
