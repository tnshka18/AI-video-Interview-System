# AI-video-Interview-System

> An automated, AI-driven first-round interview platform that captures, streams, and evaluates candidate responses in real time — so recruiters can screen hundreds of candidates asynchronously without sacrificing signal quality.

---

## Table of Contents

1. [Problem Understanding](#1-problem-understanding)
2. [Architecture Overview](#2-architecture-overview)
3. [Technical Decisions & Tradeoffs](#3-technical-decisions--tradeoffs)
4. [Failure Scenarios & Edge Cases](#4-failure-scenarios--edge-cases)
5. [Recovery Mechanisms](#5-recovery-mechanisms)
6. [Product Thinking](#6-product-thinking)
7. [Scalability Considerations](#7-scalability-considerations)
8. [Observability & Debugging](#8-observability--debugging)
9. [AI Usage Documentation](#9-ai-usage-documentation)
10. [Demo & Walkthrough](#10-demo--walkthrough)

---

## 1. Problem Understanding

### What problem are we solving?

Manual first-round interviews are expensive, time-consuming, and impossible to scale. A recruiter can conduct maybe 8–10 interviews per day. When a job posting attracts hundreds of applicants, this creates a serious bottleneck — valuable candidates get deprioritized simply due to scheduling constraints.

### Why is this system needed?

This platform automates the first screening round entirely. An AI interviewer verbally asks questions, the candidate responds on video, and the system captures, transcribes, and evaluates every response asynchronously. Recruiters receive a structured report with video playback, transcripts, and behavioral signals — all without scheduling a single call.

**Key goals:**
- Allow recruiters to screen 100s of candidates in parallel
- Maintain high-fidelity records (video + audio + transcript) of every response
- Flag integrity concerns (tab-switching, face absence) in real time
- Enable candidates to complete interviews on their own schedule

---

## 2. Architecture Overview

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Frontend                           │
│          React / Next.js (MediaRecorder API)            │
│   Hardware Check → Interview Flow → Response Capture    │
└────────────────────┬───────────────────────────────────┘
                     │  WebSocket + chunked POST
┌────────────────────▼───────────────────────────────────┐
│                   Backend API                           │
│              Node.js / Express                          │
│    Session Management · Chunk Ingestion · Proctoring    │
└──────────┬──────────────────────────┬──────────────────┘
           │                          │
┌──────────▼──────────┐   ┌──────────▼──────────────────┐
│   Object Storage    │   │      SQS Message Queues      │
│  Cloudflare R2 / S3 │   │  AUDIO_MERGE_QUEUE           │
│  Raw media chunks   │   │  TRANSCRIPTION_QUEUE         │
└─────────────────────┘   └──────────┬───────────────────┘
                                     │
                          ┌──────────▼───────────────────┐
                          │   Lambda Workers (Python)    │
                          │  FFmpeg merge · Deepgram STT │
                          │  AI Scoring · Evaluation     │
                          └──────────────────────────────┘
```

### Media Flow

```
1. Candidate speaks → MediaRecorder API captures audio/video as Blobs
2. Blobs chunked every N seconds → sent via WebSocket or high-frequency POST
3. Backend writes chunks to R2/S3 with deterministic keys: chunk_001.wav, chunk_002.wav ...
4. AUDIO_MERGE_QUEUE triggered → Lambda runs FFmpeg to merge chunks into final file
5. Merged file enqueued to TRANSCRIPTION_QUEUE
6. Deepgram Lambda processes audio → returns transcript + word-level timestamps
7. Evaluation Orchestrator scores response → result stored in AiInterview.session_data
```

### WebSocket / Event Flow

```
Client                          Server
  │── connect (session_id) ────▶ │  Authenticate + load session state
  │◀─ session_restored ──────── │  Resume from AiInterview.session_data
  │                              │
  │── chunk_data (blob) ────────▶│  Write to S3, enqueue merge job
  │── proctoring_event ─────────▶│  Log: tab_switch / face_absent / face_multiple
  │◀─ question_next ─────────── │  AI TTS plays next question
  │◀─ proctoring_flag ──────────│  Alert: suspicious activity detected
  │                              │
  │── disconnect ───────────────▶│  Session persisted; reconnect window open
```

---

## 3. Technical Decisions & Tradeoffs

### Streaming over Full Upload

**Decision:** Media is streamed in small chunks rather than uploaded as a single file after the session ends.

**Why:** If the candidate loses connectivity mid-interview, we already have 90%+ of the captured data. A full-upload model would lose everything on disconnect. Streaming also avoids holding large video files in browser memory, which causes crashes on lower-end devices.

**Tradeoff:** Chunk reassembly adds backend complexity (FFmpeg merge step, ordering logic). This is an acceptable cost given the reliability gain.

### Asynchronous AI Processing

**Decision:** Transcription and AI scoring run via SQS + Lambda rather than inline in the API request.

**Why:** These operations take 5–30 seconds. Running them synchronously would block the question transition flow and create a poor candidate experience. Async processing keeps P99 question-transition latency under 2 seconds.

**Tradeoff:** Results are not immediately available; recruiters see a "processing" state briefly after the interview ends.

### Session State as Single Source of Truth

**Decision:** All session progress is stored in `AiInterview.session_data` — a single document that tracks current question index, chunk manifest, proctoring flags, and transcript fragments.

**Why:** This enables seamless resume-on-reconnect. The client can re-fetch state and pick up exactly where it left off, even after a full browser refresh.

**Tradeoff:** `session_data` can grow large for long interviews; periodic compaction may be needed at scale.

### MERN + Python Lambda Hybrid

**Decision:** Core API in Node.js/Express; heavy media work in Python-based Lambda functions.

**Why:** Node.js handles high-concurrency I/O well (websocket connections, chunk ingestion). Python has mature FFmpeg bindings and better AI/ML library support (Deepgram SDK, evaluation models).

---

## 4. Failure Scenarios & Edge Cases

| Scenario | Risk | Impact |
|---|---|---|
| **Network Interruption** | WebSocket drops mid-response | Partial chunk loss; session appears stalled |
| **Duplicate Chunks** | Client retries on timeout | Duplicate audio in merged file |
| **Camera / Mic Disconnect** | Browser `MediaRecorder` stops | Empty or silent chunks from that point forward |
| **Partial Upload Failure** | S3/R2 write error on individual chunk | Gap in audio reconstruction |
| **WebSocket Reconnect** | Re-handshake delay | Brief loss of proctoring signal |
| **Empty / Corrupted Chunks** | Zero-byte blob or malformed header | FFmpeg merge failure for that response |
| **Browser Crash** | Full session state lost client-side | Dependent on server-side session persistence |
| **Concurrent Question Transition** | Race condition between chunk ACK and next question signal | Question asked before previous response fully captured |

---

## 5. Recovery Mechanisms

### Session Resume on Reconnect

Every session state mutation is written to `AiInterview.session_data` before the client ACK is sent. On WebSocket reconnect, the client sends its `session_id` and the server restores:
- Current question index
- Chunks already received and their S3 keys
- Proctoring event log
- Partial transcript (if available)

The candidate resumes from the exact question they were on, with no data loss.

### Chunk Recovery Strategy

Chunks are written with zero-padded deterministic keys:

```
s3://bucket/{session_id}/responses/{question_index}/chunk_001.wav
s3://bucket/{session_id}/responses/{question_index}/chunk_002.wav
```

This means:
- FFmpeg can sort and merge chunks by filename even if they arrived out of order
- Duplicate chunks (from client retries) are detected by key collision and skipped
- Missing chunk gaps are detectable by sequence number; the system logs warnings but continues merge with available chunks

### Retry / Recovery Logic

- **Client-side:** Exponential backoff on chunk POST failures (3 retries, max 8s delay)
- **WebSocket:** Auto-reconnect with session restoration on any disconnect event
- **Lambda workers:** SQS visibility timeout ensures failed jobs are re-queued automatically; dead-letter queue captures repeated failures for manual review
- **Empty chunk detection:** Chunks below a minimum byte threshold (configurable, default 1KB) are discarded before S3 write

---

## 6. Product Thinking

### Candidate Experience

- **Hardware Check page** (mandatory pre-interview): Tests camera, microphone, and internet before the interview begins. This catches ~80% of technical issues that would otherwise disrupt the session and eliminates the "I didn't know my mic was off" failure mode.
- **Progress indicator:** Candidates see "Question X of N" so they know how much remains — reducing abandonment.
- **Reconnect grace window:** If a candidate disconnects, they have a configurable window (default: 30 minutes) to rejoin and resume without losing their session.
- **Clear question delivery:** Questions are delivered via AI TTS with a natural pause before recording begins, so candidates aren't caught off guard.

### Recruiter Experience

- **Unified review dashboard:** Resume, video playback, transcript, and AI scoring are all visible in a single panel — no tab switching between tools.
- **Proctoring summary:** Instead of reviewing raw event logs, recruiters see a high-level integrity score plus flagged timestamps (e.g., "Face absent at 2:14, tab switch at 4:32").
- **Async workflow:** Completed interviews are queued for review; recruiters can batch-process 20 interviews in the time a single live screen would take.

### Suspicious Activity Tracking

The proctoring system monitors the following signals in real time via the WebSocket channel:

| Signal | Detection Method | Threshold |
|---|---|---|
| Tab switch | `visibilitychange` browser event | Any occurrence flagged |
| Face absent | MediaPipe face detection on video stream | > 3 consecutive seconds |
| Multiple faces | Face count > 1 detected | Any occurrence flagged |
| Audio silence | RMS amplitude below threshold | > 10 consecutive seconds |
| Window blur | `blur` event on `window` | Any occurrence flagged |

Each event is timestamped and stored in `session_data.proctoring_events`. The recruiter dashboard renders these as annotated markers on the video timeline.

---

## 7. Scalability Considerations

### What May Break at Scale

- **TRANSCRIPTION_QUEUE backlog:** At 1,000 concurrent interviews, transcription jobs will queue. Current Lambda concurrency limits (default 1,000/region) may introduce delays.
- **WebSocket connection limits:** A single Node.js server handles ~10,000 concurrent WebSocket connections before memory pressure degrades performance. Horizontal scaling requires sticky sessions or a pub/sub layer (e.g., Redis Pub/Sub).
- **S3 ingress throughput:** High write rates to a single S3 prefix can throttle. Key prefix sharding by session ID distributes load across partitions.
- **`session_data` document size:** Long interviews with many proctoring events can produce large MongoDB documents; queries slow down past ~16MB.

### Future Improvements for High Concurrency

- Replace direct Lambda invocations with Step Functions for complex evaluation pipelines
- Add a Redis layer for hot session state (avoid MongoDB reads on every chunk ACK)
- Implement adaptive bitrate on the client to reduce chunk size under poor network conditions
- Shard the WebSocket layer behind an Application Load Balancer with sticky routing
- Add a CDN layer (CloudFront) for TTS audio delivery to reduce latency globally

---

## 8. Observability & Debugging

### Logging Strategy

All application events are emitted as structured JSON logs to **AWS CloudWatch**:

```json
{
  "timestamp": "2025-01-15T10:32:01Z",
  "level": "INFO",
  "service": "chunk-ingestion",
  "session_id": "abc123",
  "question_index": 2,
  "chunk_index": 7,
  "bytes": 48200,
  "s3_key": "sessions/abc123/responses/2/chunk_007.wav"
}
```

Log groups are organized by service: `chunk-ingestion`, `merge-worker`, `transcription-worker`, `evaluation-orchestrator`, `proctoring`.

### Error Tracking

- Failed chunk writes trigger a `WARN` log with the S3 error code and retry count
- FFmpeg merge failures emit an `ERROR` log with the session ID and are forwarded to the dead-letter queue
- Transcription failures (Deepgram API errors) are retried twice before moving to DLQ with full context
- CloudWatch Alarms fire on: DLQ depth > 10, Lambda error rate > 1%, P99 API latency > 3s

### Debugging Production Failures

To trace a specific interview end-to-end:

```bash
# 1. Find all logs for a session
aws logs filter-log-events \
  --log-group-name /ai-interview/chunk-ingestion \
  --filter-pattern "{ $.session_id = \"SESSION_ID\" }"

# 2. Check merge worker output
aws logs filter-log-events \
  --log-group-name /ai-interview/merge-worker \
  --filter-pattern "{ $.session_id = \"SESSION_ID\" }"

# 3. Inspect session state directly
mongo> db.aiinterviews.findOne({ session_id: "SESSION_ID" }, { session_data: 1 })
```

---

## 9. AI Usage Documentation

### How AI Tools Were Used

AI (Claude/ChatGPT) was used as a **thinking accelerator**, not a code generator. Every piece of generated code was reviewed, modified, and validated before integration.

### Specific Usage Areas

| Area | AI Role | Human Role |
|---|---|---|
| SQS queue architecture | AI proposed initial queue topology (merge queue → transcription queue → eval queue) | Reviewed tradeoffs; added DLQ strategy and retry configuration |
| FFmpeg merge logic | AI generated initial Python Lambda with chunk sorting | Debugged ordering edge cases; added empty-chunk detection |
| TTS provider selection | Used AI to compare Deepgram vs. AssemblyAI vs. Whisper on latency, cost, accuracy | Made final call based on Deepgram's real-time streaming API fit |
| Proctoring event schema | AI suggested initial event structure | Extended with severity levels and threshold configuration |
| Evaluation Orchestrator | AI drafted initial scoring rubric structure | Manually defined all scoring criteria and weights |

### Prompt Strategy

Prompts followed an **"Understand → Explore → Decide"** pattern:

1. **Understand:** *"Explain the tradeoffs between streaming media chunks via WebSocket vs. high-frequency HTTP POST for a video interview system with unreliable candidate connections."*
2. **Explore:** *"What are three different approaches to chunk ordering when chunks may arrive out of sequence from a browser MediaRecorder?"*
3. **Decide:** *"Given these constraints [listed], which approach minimizes data loss while keeping backend complexity manageable? Justify your recommendation."*

### What Was Mine vs. AI-Assisted

- **Mine:** System architecture decisions, product UX flow, proctoring signal selection, evaluation scoring criteria, database schema, all debugging and production hardening
- **AI-assisted:** Initial code scaffolding for Lambda workers, boilerplate for SQS consumer patterns, FFmpeg subprocess invocation syntax
- **AI-generated + heavily modified:** WebSocket reconnect logic, chunk deduplication strategy

---

