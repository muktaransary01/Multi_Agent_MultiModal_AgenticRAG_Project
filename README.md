# BRAND GUARDIAN AI - ARCHITECTURE DIAGRAMS

## Diagram 1: High-Level System Architecture

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃                   BRAND GUARDIAN AI SYSTEM ARCHITECTURE               ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛

                            USER/CLIENT LAYER
                    ┌─────────────────────────────┐
                    │  Web/Mobile/REST API Call   │
                    │  POST /audit                │
                    │  {video_url: "..."}         │
                    └──────────────┬──────────────┘
                                   │
                                   ↓

                    ╔════════════════════════════════╗
                    ║    FASTAPI SERVER (Port 8000)  ║
                    ╠════════════════════════════════╣
                    ║  • URL validation              ║
                    ║  • Rate limiting               ║
                    ║  • Request routing             ║
                    ║  • Response formatting         ║
                    ╚════════════════════╤═══════════╝
                                        │
                                        ↓
                    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
                    ┃  LANGGRAPH StateGraph         ┃
                    ┃  (Agentic Workflow Engine)    ┃
                    ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
                    ┃                               ┃
                    ┃  ┌─────────────────────────┐  ┃
                    ┃  │  Node 1: index_video    │  ┃
                    ┃  │  ◆ Download (yt-dlp)    │  ┃
                    ┃  │  ◆ Upload (Azure VI)    │  ┃
                    ┃  │  ◆ Poll Status (30s)    │  ┃
                    ┃  │  ◆ Extract Text         │  ┃
                    ┃  │  ⏱ ~100-200 seconds     │  ┃
                    ┃  └──────────┬──────────────┘  ┃
                    ┃             │                  ┃
                    ┃             ↓                  ┃
                    ┃  ┌─────────────────────────┐  ┃
                    ┃  │ Node 2: audit_content   │  ┃
                    ┃  │ ◆ Build RAG Query       │  ┃
                    ┃  │ ◆ Embed Text (OpenAI)   │  ┃
                    ┃  │ ◆ Search KB (k=3)       │  ┃
                    ┃  │ ◆ Call GPT-4 (T=0.0)    │  ┃
                    ┃  │ ◆ Parse Violations      │  ┃
                    ┃  │ ⏱ ~2-5 seconds          │  ┃
                    ┃  └──────────┬──────────────┘  ┃
                    ┃             │                  ┃
                    ┃         Final State            ┃
                    ┃      (Violations Array)       ┃
                    ┗━━━━━━━━━━━━╤━━━━━━━━━━━━━━━┛
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ↓                ↓                ↓
        ┌───────────────┐  ┌──────────────┐  ┌──────────────┐
        │ EXTERNAL:     │  │ EXTERNAL:    │  │ EXTERNAL:    │
        │ YouTube       │  │ Azure        │  │ OpenAI       │
        │               │  │ Services     │  │ + Search     │
        │ ◆ Video       │  │              │  │              │
        │   Download    │  │ ◆ Video      │  │ ◆ GPT-4      │
        │   (yt-dlp)    │  │   Indexer    │  │ ◆ Embeddings │
        │               │  │ ◆ Monitor    │  │ ◆ Search     │
        │ Response:     │  │   (Telemetry)  │ ◆ Alerts     │
        │ MP4 Video     │  │              │  │              │
        └───────────────┘  └──────────────┘  └──────────────┘

```
---

## Diagram 2: Step-by-Step Request Flow (Detailed)

```
USER                    FASTAPI              LANGGRAPH              AZURE SERVICES
 |                         |                     |                        |
 |-- POST /audit --------> |                     |                        |
 |   {video_url: "..."}    |                     |                        |
 |                         |                     |                        |
 |                    [VALIDATE]                 |                        |
 |                    - URL format               |                        |
 |                    - Rate limits              |                        |
 |                    - Build VideoAuditState    |                        |
 |                         |                     |                        |
 |                         |-- workflow.invoke-->|                        |
 |                         |                     |                        |
 |                         |           [NODE 1: index_video_node]         |
 |                         |           ⏱ ~150-300 seconds total           |
 |                         |                     |                        |
 |                         |                     |-- yt-dlp download ---> YouTube
 |                         |                     |   (30-60s)             |
 |                         |                     |<-- video.mp4 --------- |
 |                         |                     |                        |
 |                         |                     |-- POST /upload ------> Azure Video Indexer
 |                         |                     |   (20-40s)             |
 |                         |                     |<-- video_id="vi-..." --+
 |                         |                     |                        |
 |                         |                     |-- GET /status (poll) ->| [Running ML]
 |                         |                     |   every 30s            | • Speech-to-Text
 |                         |                     |   until "Processed"    | • OCR
 |                         |                     |   (100-200s)           | • Speaker ID
 |                         |                     |<-- transcript, ocr ----+
 |                         |                     |                        |
 |                         |           [NODE 2: audit_content_node]       |
 |                         |           ⏱ ~2-5 seconds total               |
 |                         |                     |                        |
 |                         |                     |-- embed text --------> Azure OpenAI
 |                         |                     |   (100-200ms)          | (text-embedding-3-small)
 |                         |                     |<-- 1536-dim vector ----+
 |                         |                     |                        |
 |                         |                     |-- cosine search -----> Azure AI Search
 |                         |                     |   k=3 (500-800ms)      | index: compliance-rules
 |                         |                     |<-- top 3 rule docs ----+
 |                         |                     |                        |
 |                         |                     |-- GPT-4 prompt ------> Azure OpenAI GPT-4
 |                         |                     |   temp=0.0 (1-3s)      | [Rules + Transcript]
 |                         |                     |<-- violations JSON ----+
 |                         |                     |                        |
 |                         |                     | [PARSE JSON]           |
 |                         |                     | Build ComplianceIssue  |
 |                         |                     | Append to state        |
 |                         |                     |                        |
 |                         |<-- Final State ------+                        |
 |                         |   compliance_results filled                  |
 |                         |                     |                        |
 |                    [BUILD RESPONSE]            |                        |
 |                    HTTP 200 + violations JSON  |                        |
 |                         |                     |                        |
 |<-- HTTP 200 JSON --------+                     |                        |
 |   {violations:[...]}    |                     |                        |


TOTAL TIME: ~175-335 seconds (2.9 to 5.6 minutes)
─────────────────────────────────────────────────────────────────────────────
 STEP                        TIME        % OF TOTAL
─────────────────────────────────────────────────────────────────────────────
 1. YouTube Download          30-60s          18%
 2. Azure VI Upload           20-40s          12%
 3. Azure VI ML Processing   100-200s    *** 55% (BOTTLENECK) ***
 4. OpenAI Embed              ~200ms           0%
 5. AI Search (k=3)           ~700ms           0%
 6. GPT-4 Call                 1-3s            1%
─────────────────────────────────────────────────────────────────────────────
```

### State at Each Stage

```
INITIAL STATE (after STEP 2 - FastAPI)
  {
    video_url:          "https://youtube.com/watch?v=dQw4w9WgXcQ",
    audit_id:           "a1b2c3d4-...",
    transcript:         "",          <-- empty
    ocr_text:           "",          <-- empty
    compliance_results: []           <-- empty
  }

AFTER NODE 1 (index_video_node)
  {
    video_url:          "https://youtube.com/watch?v=dQw4w9WgXcQ",
    video_id:           "vi-abc123xyz",
    transcript:         "Hello everyone, today we're discussing skincare...",
    ocr_text:           "Paid partnership | Use code SAVE50 | Limited time",
    compliance_results: []           <-- still empty
  }

AFTER NODE 2 (audit_content_node)
  {
    video_url:          "https://youtube.com/watch?v=dQw4w9WgXcQ",
    video_id:           "vi-abc123xyz",
    transcript:         "Hello everyone...",
    ocr_text:           "Paid partnership...",
    compliance_results: [            <-- NOW FILLED
      { rule: "FTC_DISCLOSURE",  severity: "HIGH",   timestamp: "00:45" },
      { rule: "LINK_DISCLOSURE", severity: "MEDIUM",  timestamp: "02:15" }
    ]
  }

FINAL HTTP RESPONSE (HTTP 200)
  {
    "audit_id":               "a1b2c3d4-e5f6-47a8-9b1c-2d3e4f5a6b7c",
    "status":                 "completed",
    "violations": [
      {
        "rule":      "FTC_DISCLOSURE",
        "severity":  "HIGH",
        "evidence":  "No clear disclosure in first 5 seconds. Appears at 0:45.",
        "timestamp": "00:45"
      },
      {
        "rule":      "LINK_DISCLOSURE",
        "severity":  "MEDIUM",
        "evidence":  "'Use code SAVE50' shown but not labeled as affiliate link.",
        "timestamp": "02:15"
      }
    ],
    "processing_time_seconds": 175
  }
```

---

## Diagram 3: Performance Breakdown

```
╔════════════════════════════════════════════════════════════════════════╗
║                   LATENCY ANALYSIS & BOTTLENECK                        ║
╚════════════════════════════════════════════════════════════════════════╝

TOTAL END-TO-END TIME: 2.9 - 5.6 minutes (175-335 seconds)

┌──────────────────────────────────────────────────────────────────────┐
│ Node 1: index_video_node                                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Segment A: Download YT Video       30-60s        ▓▓ (10-20%)         │
│ Segment B: Upload to Azure VI      20-40s        ▓▓ (6-13%)          │
│ Segment C: Poll & Process          100-200s      ▓▓▓▓▓ (50-65%)      │
│                                     ────────────────────────────     │
│ Subtotal:                           ~150-300s     (65-70% of E2E)     │
│                                                                      │
│ WHY C IS THE BOTTLENECK:                                             │
│ • Azure Video Indexer runs ML models in parallel                     │
│ • Speech-to-Text processing        (depends on video length)        │
│ • OCR processing                   (depends on content density)      │
│ • Speaker Diarization              (computational intensive)         │
│ • Scene Detection                  (frame-by-frame analysis)         │
│ → This is MANAGED SERVICE overhead - Cannot optimize further       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ Node 2: audit_content_node                                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Phase A: Embed Query               100-200ms     ▓ (3%)              │
│ Phase B: Search Knowledge Base     500-800ms     ▓▓ (20%)            │
│ Phase C: Call GPT-4                1-3s          ▓▓ (40%)            │
│ Phase D: Parse Response            50-100ms      ▓ (2%)              │
│                                     ───────────────────────────      │
│ Subtotal:                          ~2-4s         (1-2% of E2E)       │
│                                                                      │
│ NOTE: This is FAST because:                                          │
│ • Text embeddings are lightweight                                    │
│ • Vector search is optimized                                         │
│ • GPT-4 is fast for compliance patterns                              │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

COST BREAKDOWN (per audit):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Service               Cost per Audit        Notes
  ─────────────────────────────────────────────────────────────────
  Azure Video Indexer   .05 - .50        Depends on video length
  OpenAI Embeddings     ~.00002            Fixed API call
  GPT-4 Inference       .02 - .05        Token usage varies
  Azure AI Search       ~.00001            Query cost minimal
  ─────────────────────────────────────────────────────────────────
  TOTAL                 .08 - .56        Most cost from VI

```
---

## Diagram 4: System Component Dependencies

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃               COMPONENT INTERACTION MATRIX                          ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛

FastAPI Server
  │
  ├─→ LangGraph StateGraph
  │    │
  │    ├─→ index_video_node
  │    │    ├─→ yt-dlp              (Download video)
  │    │    └─→ Azure Video Indexer (Process video)
  │    │         ├─→ STT            (Speech to Text)
  │    │         ├─→ OCR            (Text extraction)
  │    │         ├─→ Speaker ID     (Diarization)
  │    │         └─→ Scene Detect   (Boundaries)
  │    │
  │    └─→ audit_content_node
  │         ├─→ OpenAI Embeddings   (text-embedding-3-small)
  │         ├─→ Azure AI Search     (Vector DB lookup)
  │         ├─→ OpenAI GPT-4        (Compliance reasoning)
  │         └─→ JSON Parser         (Response extraction)
  │
  └─→ Azure Monitor                 (Telemetry & Alerts)

```
