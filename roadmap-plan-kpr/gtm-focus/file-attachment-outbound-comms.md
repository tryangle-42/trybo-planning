# File Attachment for Outbound Communications

## Context

When a user uploads a file in the chat UI, the system should store that file in an optimised way in a storage bucket and make its URL path available to the agent. Whenever the agent decides to include that file in any outbound communication — WhatsApp, Slack, email, or any other channel — the agent passes the file URL to that channel. The communication channel is then responsible for figuring out how to handle and deliver that file in its own way.

The storage bucket is **private** — file access is authenticated using the user's session access token, the same way all other APIs are authenticated via Supabase. No public URLs.

The file should persist as long as the chat exists. If the user deletes the chat, the file is cleaned up. No separate permanent storage is needed beyond the chat lifecycle.

Text extraction or content parsing from the uploaded file is handled by a separate component — this component is only concerned with storing the binary and making its URL available.

The file metadata (doc_id, filename, mime_type, storage_path) lives as part of **chat memory** — the same place and same chat_id scope where other chat data lives. When the agent loads chat memory for a conversation, the uploaded file references are already there. No separate store, no separate query. The agent sees them the same way it sees any other chat context.

### Goals

1. **Optimised file storage** — store uploaded files efficiently in a cloud storage bucket
2. **File metadata in chat memory** — file references live under the chat_id in chat memory, discoverable by the agent alongside all other chat context
3. **Channel-agnostic handoff** — the agent passes the URL to any communication channel; the channel handles delivery
4. **Chat-scoped lifecycle** — files live as long as the chat, cleaned up on chat deletion

### Supported File Types

| Category | Formats |
|----------|---------|
| **Text documents** | PDF, DOCX, TXT |
| **Tabular data** | CSV, XLS, XLSX |
| **Audio** | MP3, WAV, M4A, OGG |

All file types are treated the same by this component — store the binary, return a URL.

---

## Component Responsibility

```
┌─────────────────────────────────────────────────────────┐
│              THIS COMPONENT'S RESPONSIBILITY             │
│                                                          │
│  On upload:                                              │
│  1. User uploads file in chat                            │
│  2. Validate file type and size                          │
│  3. Store original binary in storage bucket              │
│  4. Write file metadata into chat memory under chat_id   │
│     (doc_id, filename, mime_type, storage_path)          │
│     → agent sees it when it loads chat context           │
│                                                          │
│  On removal (cross button before send):                  │
│  5. Delete binary from bucket                            │
│  6. Remove file metadata from chat memory                │
│                                                          │
│  On chat deletion:                                       │
│  7. Delete all file binaries from bucket for that chat   │
│  8. Remove all file metadata from chat memory            │
│                                                          │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ↓  (file URL)
┌─────────────────────────────────────────────────────────┐
│              NOT THIS COMPONENT'S RESPONSIBILITY         │
│                                                          │
│  - Text extraction / content parsing from the file       │
│    → handled by a separate component                     │
│                                                          │
│  - How each channel delivers the file                    │
│    → each channel figures it out on its own              │
│                                                          │
│  - Agent's decision to attach or not attach              │
│    → agent's own judgment based on context               │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## How It Should Work

### Upload & Storage

```
User picks a file in the chat UI (any supported type)
  → File uploaded to backend along with the current chat_id
  → Backend validates file type and size
  → Store the original binary in a private Supabase Storage bucket
      under a user-scoped path for isolation
  → Persist file metadata (doc_id, filename, mime_type, storage_path)
      in chat memory under the chat_id
  → Return doc_id and storage path to the frontend
  → File access is authenticated via user session token (same as all other APIs)
  → Frontend shows the file as a chip in the message composer
  → Agent automatically sees the file when it loads chat memory
```

### How the Agent Sees Files

```
Agent loads chat memory for a conversation
  → File metadata is already there as part of chat memory (same chat_id scope)
  → No separate query or discovery mechanism needed
  → Agent sees: doc_id, filename, mime_type, storage_path for each uploaded file
  → Agent decides whether to include files based on user's request
  → When agent decides to send via any channel, it passes the storage path
  → The channel receives the path and handles delivery on its own
```

### Attachment Tracking in Chat Messages

```
User sends a chat message with one or more attached files
  → Message payload includes attachment_ids (list of doc_ids)
  → Backend resolves each attachment_id to its metadata
      (filename, mime_type, storage_path)
  → Stores attachment references in the message's metadata
  → Files are now discoverable as part of the conversation
```

### Pre-Send Removal

```
User uploads a file → file appears as a chip in the input box with a cross (X) button
  → User taps the cross button before sending the message
  → File is deleted from the storage bucket
  → File metadata record is deleted
  → Chip is removed from the input box
  → As if the file was never uploaded
```

This is important because upload happens immediately (not on message send), so if the user changes their mind, the file must be fully cleaned up — not just hidden from the UI but actually removed from storage and metadata.

### File Lifecycle

```
File uploaded → metadata stored in chat memory under chat_id
  → File persists as long as the chat exists
  → User deletes chat → cascade cleanup:
      1. Find all file records in chat memory for this chat
      2. Delete from storage bucket
      3. Delete metadata from chat memory
      4. Proceed with chat/message deletion

User removes file before sending (cross button)
  → Immediate cleanup: delete from bucket + delete from chat memory

Files uploaded outside a chat context
  → Persist until explicitly deleted by the user
```

---

## Design Phases

### Phase 1: Binary Storage in Bucket

Store the original uploaded file binary in a private Supabase Storage bucket. A `chat-attachments` bucket holds files, scoped by user ID for isolation. The bucket is private — access is authenticated using the user's session access token, the same Supabase auth mechanism used across all APIs.

File metadata (doc_id, filename, mime_type, storage_path) is stored in chat memory under the chat_id. This means the agent automatically sees the file references when it loads chat context — no separate discovery mechanism needed.

All file types are handled identically — validate, store binary, persist metadata in chat memory. No content processing happens in this component.

---

### Phase 2: Attachment Tracking in Chat Messages

The chat message includes `attachment_ids` — a list of doc_ids. The backend resolves each to its metadata from chat memory and stores attachment references (doc_id, filename, mime_type, storage_path) in the message metadata. Since file metadata already lives in chat memory under the chat_id, the agent doesn't need a separate endpoint to discover files — they're part of the chat context it already loads.

---

### Phase 3: Agent Awareness

Since file metadata lives in chat memory, the agent already sees the uploaded files when it loads chat context. The planner prompt includes the available files and instructs the agent that it can pass file storage paths to communication channels. Communication skill schemas include an optional input for file references.

The agent decides — it's informed about available files but uses its own judgment. The existing approval gate ensures the user confirms before anything goes out.

---

### Phase 4: Frontend Wiring

The upload function passes `chat_id` so files are linked to the chat. Response includes the storage path. When sending a message, `attachment_ids` are included in the payload. The file picker allows selecting all supported file types (text documents, tabular data, audio).

Each uploaded file appears as a chip in the input box with a cross (X) button. Tapping the cross button before sending the message triggers a full cleanup — deletes the file from the storage bucket, removes the metadata record, and removes the chip from the UI. Since files are uploaded immediately (not deferred to message send), this pre-send removal ensures no orphaned files are left behind when the user changes their mind.

---

### Phase 5: Chat Deletion Cascade

When a chat is deleted: find all file records in chat memory for that chat, delete from the storage bucket, delete metadata from chat memory, then proceed with chat/message cleanup. Files without a chat link persist until explicitly deleted.

---

## Implementation Priority & Time Estimates

All estimates assume AI-assisted development (Claude). Coding time is reduced compared to manual, but testing, device builds, and verification time remain as-is.

**Working day = 8 hours**

### Phase 1: Binary Storage in Bucket
*Dependencies: None*

| Task | Hours |
|------|-------|
| Implement bucket setup, config, upload-to-storage flow | 2 |
| Expand allowed file types (tabular, audio) | 0.5 |
| Update metadata to store storage_path + chat_id | 0.5 |
| Unit tests | 1 |
| Manual verification (files landing in bucket correctly) | 0.5 |
| **Phase 1 Total** | **4.5** |

### Phase 2: Attachment Tracking in Chat Messages
*Dependencies: P1*

| Task | Hours |
|------|-------|
| Update chat message schema to accept attachment_ids | 0.5 |
| Resolve attachment_ids to metadata, store in message | 1 |
| Endpoint to list all attachments in a chat | 0.5 |
| Unit tests | 0.5 |
| Manual verification | 0.5 |
| **Phase 2 Total** | **3** |

### Phase 3: Agent Awareness
*Dependencies: P1, P2*

| Task | Hours |
|------|-------|
| Add available_attachments to execution context | 0.5 |
| Populate from chat's uploaded files | 1 |
| Update planner prompt to list available files | 1 |
| Update communication skill schemas with optional file input | 0.5 |
| Resolve attachment references in skill adapter | 0.5 |
| Unit tests | 1 |
| Integration testing (run agent, verify it discovers and passes files) | 1 |
| **Phase 3 Total** | **5.5** |

### Phase 4: Frontend Wiring
*Dependencies: P1, P2*

| Task | Hours |
|------|-------|
| Upload function passes chat_id, response includes storage path | 0.5 |
| Add attachment_ids to chat message payload | 0.5 |
| Expand file picker to allow all supported types | 0.5 |
| Pre-send removal: cross button on chip → delete from bucket + metadata | 1.5 |
| Unit tests | 0.5 |
| Android build + on-device testing (upload, attach, remove, send — all file types) | 1.5 |
| iOS build + on-device testing (same test cases) | 1.5 |
| Platform-specific bug fixes | 1 |
| **Phase 4 Total** | **7.5** |

### Phase 5: Chat Deletion Cascade
*Dependencies: P1*

| Task | Hours |
|------|-------|
| Implement cascade: find linked files → delete from bucket → delete metadata | 1 |
| Unit tests | 0.5 |
| Manual verification (delete chat, confirm files are gone from bucket) | 0.5 |
| **Phase 5 Total** | **2** |

### End-to-End Integration Testing
*After all phases complete*

| Task | Hours |
|------|-------|
| Full flow test on Android (upload → attach → agent picks up → channel receives URL → delete chat cleanup) | 1 |
| Full flow test on iOS (same) | 1 |
| Edge cases: remove before send, multiple files, mixed file types, large files | 1 |
| Bug fixes from integration testing | 1.5 |
| **Integration Total** | **4.5** |

**Recommended order:** P1 (day 1 morning) → P2 + P4 in parallel (day 1 afternoon + day 2) → P3 (day 2-3) → P5 (day 3) → Integration testing (day 3-4)

Note: P2 and P4 can be worked on in parallel since they're backend schema + frontend wiring respectively, but device builds (Android + iOS) in P4 take wall-clock time even when the coding is fast.

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Supabase Storage for binaries** | Already used for other file storage in the system. Same infra, no new dependencies. |
| **Private bucket + session token auth** | File access uses the same user session token that authenticates all other APIs. No public URLs. Consistent auth model, no separate access mechanism. The communication channel (being a backend service) accesses the file using the user's session context it already has. |
| **All file types stored identically** | This component doesn't parse or extract content — it just stores binaries. Text extraction is a separate component's job. |
| **File metadata in chat memory** | File references live in chat memory under the chat_id, same as other chat data. Agent sees them automatically when loading chat context. No separate store, no separate query. |
| **Chat-scoped lifecycle** | Files persist with the chat. Deletion cascades. No permanent storage beyond the chat's lifetime. |
| **Channel-agnostic handoff** | This component only stores and serves the path. Each channel figures out delivery on its own. Clean separation of concerns. |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Session token expiry during delivery | Channel can't fetch file if token expired mid-flow | Backend services operate within the user's session context; token refresh is handled at the session layer as with all other API calls. |
| Large files slow upload | User perceives delay during upload | 10 MB default limit is safe. Increase later if needed. |
| Agent passes URL to a channel that can't handle that file type | Delivery fails at channel level | Channel should validate and return a clear error. Agent can retry with a different approach. |
| Storage costs grow unbounded | Cloud bill surprise | Chat deletion cascade cleans up. Monitor bucket size. Optional retention policy for orphans. |
| File uploaded but chat_id not linked | Orphaned file after chat deletion | Enforce chat_id at upload time when uploading from within a chat. Standalone uploads have explicit delete. |

---

## Total Effort

| Phase | Hours | Days (8h/day) |
|-------|-------|---------------|
| P1 — Binary storage in bucket | 4.5 | 0.5 |
| P2 — Attachment tracking in chat messages | 3 | 0.5 |
| P3 — Agent awareness | 5.5 | 0.5 |
| P4 — Frontend wiring + device testing | 7.5 | 1 |
| P5 — Chat deletion cascade | 2 | 0.25 |
| End-to-end integration testing (Android + iOS) | 4.5 | 0.5 |
| **Total** | **27 hours** | **~3.5 days** |
