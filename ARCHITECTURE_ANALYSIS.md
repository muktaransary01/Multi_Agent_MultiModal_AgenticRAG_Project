# BRAND GUARDIAN AI - ARCHITECTURE ANALYSIS

## AI ARCHITECT INTERVIEW Q&A

This document addresses 12 critical architecture questions you'd face in an enterprise interview.

---

### Q1: Why LangGraph Instead of Airflow, Step Functions, or Event Bus?

**Context:** The workflow has 2 sequential steps: Index Video → Audit Content

**Answer:**

LangGraph was chosen for 4 reasons:

1. **Deterministic DAG Execution**
   - Step Functions: Requires JSON DSL, hard to test locally
   - Airflow: Requires Airflow server, more operational overhead
   - LangGraph: Python-native, testable in-process, portable

2. **State Management**
   - Single source of truth (VideoAuditState TypedDict)
   - Operator.add for append-only violation accumulation (no overwrites)
   - Automatic state threading through nodes

3. **Ecosystem Integration**
   - Built-in LangSmith observability and tracing
   - First-class LangChain integration (no wrapper needed)
   - Easy to add agents/tools later

4. **Local Development**
   - No server required—run locally with .invoke()
   - Real-time debugging without Docker/Kubernetes
   - Faster iteration cycle

**Trade-off:** Vendor lock-in to LangChain ecosystem (vs vendor-neutral Airflow)

**Production Path:** Can compile to Step Functions DSL later if needed

---

### Q2: The Biggest Problem I See: Synchronous Blocking API

**Current Architecture:**
`
HTTP POST /audit
  ↓
workflow.invoke()  ← BLOCKS for 3-4 minutes
  ↓
HTTP 200 + results
`

**The Problem:**

- FastAPI worker thread blocked during entire 3-4 minute audit
- If HTTP client timeout < 4 min (common: 30-60s), request dies without knowing if audit completed
- Cannot handle concurrent requests efficiently
- Example: 2 concurrent audits block both worker threads; 3rd request queues

**Current Code (Backend/src/api/server.py, lines 127-200):**
`python
@app.post("/audit")
def audit_endpoint(request: AuditRequest):
    result = workflow.invoke({  # ← THIS BLOCKS
        "video_url": request.video_url
    })
    return result
`

**The Fix (Async Pattern):**
`python
@app.post("/audit")
async def audit_endpoint(request: AuditRequest):
    task_id = str(uuid4())
    background_tasks.add_task(
        run_workflow_background, 
        task_id, 
        request.video_url
    )
    return {"task_id": task_id, "status": "queued"}

# Polling endpoint
@app.get("/audit/{task_id}")
async def get_audit_status(task_id: str):
    result = await db.get_audit_result(task_id)
    return result
`

**Implementation Options:**
1. **FastAPI background_tasks + polling** (simple, good for <10s tasks)
2. **Azure Queue Storage + Functions** (scalable, event-driven)
3. **Celery + Redis** (overkill for this scale)

**Recommendation:** Use FastAPI background tasks initially, migrate to Azure Queue later when you hit concurrency limits.

---

### Q3: How Does RAG Work in This System?

**The Flow:**

1. **Retrieval Phase** (backend/src/graph/nodes.py, lines 70-100)
   `
   Transcript + OCR
        ↓
   Combine into query: "Validate video for FTC and YouTube compliance"
        ↓
   Embed query (text-embedding-3-small, 1536 dims)
        ↓
   Azure AI Search (cosine similarity, k=3 docs)
        ↓
   Get top 3 compliance rule documents
   `

2. **Augmentation Phase** (lines 100-130)
   `
   System Prompt:
   "You are a compliance auditor. Use these rules:
   {RETRIEVED_RULES}
   
   Audit this video:
   {TRANSCRIPT}
   {OCR}
   
   Return JSON with violations."
   `

3. **Inference Phase** (lines 130-160)
   `
   Call GPT-4 (temp=0.0 for determinism)
   Response: 
   {
     "violations": [
       {"rule": "FTC_DISCLOSURE", "severity": "HIGH", "evidence": "..."}
     ]
   }
   `

**Knowledge Base Details:**
- Indexed in Azure AI Search under "compliance-rules" index
- Documents: FTC Influencer Guide + YouTube Ad Policy PDFs
- Chunked by section (10-15 pages per doc)
- Each chunk embedded with text-embedding-3-small

**RAG Quality Issues:**

⚠️ **Problem 1: No Source Attribution**
- LLM cites "Rule XYZ" but we don't verify it exists in KB
- Could hallucinate rules that don't exist
- Fix: Add source document ID to every violation

⚠️ **Problem 2: K=3 May Be Insufficient**
- If all 3 docs are low-confidence matches, audit is garbage
- Example: Query "sponsorship disclosure" might not match if KB uses "paid partnership"
- Fix: Dynamic k based on query similarity score (k=5 if all matches <0.7, k=3 if >0.8)

⚠️ **Problem 3: No Confidence Scoring**
- LLM doesn't tell us "I'm 92% sure this is a violation"
- Can't distinguish HIGH vs LOW confidence violations
- Fix: Prompt GPT-4 to return confidence_score (0-1) per violation

⚠️ **Problem 4: KB Not Versioned**
- Same video audited twice in different weeks might get different results
- No way to track KB evolution
- Fix: Version every KB document with timestamp, store KB version in audit record

---

### Q4: Why Use Azure Video Indexer? Why Not Self-Hosted?

**Comparison:**

| Aspect | Azure VI | Self-Hosted (e.g., Whisper) |
|--------|----------|---------------------------|
| **Cost** | \.05-0.50/video | \ (self-hosted) |
| **Setup Time** | 5 min (SDK) | 2-3 days (build pipeline) |
| **Accuracy** | 99%+ (enterprise models) | 92-96% (open-source) |
| **OCR** | Built-in (including handwriting) | Requires separate service |
| **Speaker Diarization** | Yes (distinguish speakers) | No (basic) |
| **Maintenance** | None | Full ops burden |
| **Scale** | Auto (Azure managed) | Manual provisioning |
| **Model Updates** | Automatic | Manual retraining |

**Why VI Was Chosen:**

1. **Transcript Quality**
   - VI: ~99% accuracy (trained on 1M+ hours)
   - Whisper: ~94% accuracy (trained on 680K hours)
   - For compliance: 5% difference in accuracy is material (false negatives = regulatory risk)

2. **OCR Capability**
   - VI: Extracts overlays, captions, chyrons, thumbnails
   - Whisper: No OCR
   - FTC violations often in graphic text, not speech

3. **Speaker Identification**
   - VI: Identifies who said what (brand ambassador vs influencer)
   - Whisper: No speaker info
   - Critical for disclosure audits (must know who disclosed)

**Cost Justification:**
- 1000 audits/month × \.25 average = \/month
- Alternative: 2 engineers spending 1 day/month on Whisper pipeline = \,000/month
- VI pays for itself 10x over

**Bottleneck Reality:**
- VI processing is 50% of E2E latency (100-200s for 10-min video)
- This is NOT optimizable—it's ML inference time
- Switching to Whisper would only save \.25/video, still blocked on same compute time

---

### Q5: How Do You Handle Compliance Rule Conflicts?

**Scenario:** FTC says "disclose within 5 seconds" but YouTube says "disclose in opening 3 seconds"

**Current Implementation:** No conflict handling—both rules are in KB, GPT-4 reasons about both

**Problem:**
- No priority/hierarchy
- No versioning of conflicting rules
- If KB has both FTC and YouTube rules, unclear which takes precedence

**Solution (Recommended):**

1. **Explicit Rule Hierarchy**
   `python
   RULE_PRIORITY = {
       "FTC_DISCLOSURE": {"priority": 1, "jurisdiction": "US", "source": "16 CFR 255"},
       "YOUTUBE_MONETIZATION": {"priority": 2, "jurisdiction": "YouTube", "source": "YPP Policy"}
   }
   `

2. **Store in KB with Metadata**
   `
   Document in AI Search:
   {
       "rule_id": "FTC_DISCLOSURE",
       "text": "All material connections must be disclosed...",
       "priority": 1,
       "effective_date": "2023-01-01",
       "conflict_with": ["YOUTUBE_MONETIZATION"],
       "resolution": "FTC takes precedence for US content"
   }
   `

3. **Prompt GPT-4 with Conflict Guidance**
   `
   "If both FTC and YouTube rules apply, FTC takes precedence.
   Flag as 'CONFLICT_FTC_PRIORITY' if there's a difference."
   `

**In Production:**
- Maintain rule conflict registry
- Update whenever new rules added
- Test with known conflicting scenarios before deployment

---

### Q6: Why Temperature=0.0 for GPT-4?

**Setting:** Azure OpenAI GPT-4, temperature=0.0

**What This Means:**
- Temperature: controls randomness in LLM output (0.0 = deterministic, 2.0 = creative)
- At 0.0: Same input always produces same output
- At 0.7 (default): Same input produces different outputs each time

**Why This Matters for Compliance:**
- **Requirement 1: Reproducibility**
  - Same video audited twice = same violations flagged
  - Legal/audit trail demands this
  - Temperature=0.7 would fail: might flag violation on run 1, miss it on run 2

- **Requirement 2: Defensibility**
  - If violation is contested, we can replay audit deterministically
  - Prove the same data → same output
  - Temperature=0.7 would be indefensible (randomness in reasoning)

- **Requirement 3: Testing**
  - Can write unit tests for compliance logic
  - Test case: Input video X should flag violations [Y, Z]
  - With temperature=0.7, test fails 30% of the time randomly

**Trade-off:**
- **Pro:** Deterministic, testable, reproducible
- **Con:** Less creative/nuanced reasoning
  - Example: Edge case ambiguity → might miss creative interpretation
  - But this is actually good for compliance (strict interpretation is safer)

**Alternative Considered:**
- Temperature=0.3 (compromise): more predictable than 0.7, but not 100% deterministic
- Decision: 0.0 better for compliance domain (clarity > creativity)

---

### Q7: Latency Breakdown—Why 3-4 Minutes?

**Total E2E: 175-335 seconds** (10-min video)

| Phase | Time | % of Total | Bottleneck? |
|-------|------|-----------|------------|
| YouTube DL | 30-60s | 20% | Network |
| VI Upload | 20-40s | 10% | Network |
| VI Processing | 100-200s | 50% | **YES** |
| RAG Retrieval | 0.8s | <1% | No |
| LLM Inference | 1-3s | 2% | No |
| JSON Parse | 0.2s | <1% | No |

**The 50% Bottleneck: Azure Video Indexer Processing**

VI doesn't immediately return results. Flow:
1. Upload video (5s)
2. VI starts processing job (async)
3. Poll every 30s: "Is it done yet?"
4. After 100-200s: Returns "Processed"
5. Extract transcript, OCR, metadata

Why so long? VI is running:
- Speech recognition (Whisper-class model)
- Speaker diarization (identify different speakers)
- OCR on every frame (extract text overlays)
- Scene detection (split into topics)
- Face recognition (identify people)
- All in parallel on GPU

**Can We Speed It Up?**

❌ **Not really.** This is ML compute time, not network overhead.

Options:
1. **Use faster VI tier?**
   - VI only has one processing tier
   - All customers get same speed
   
2. **Use Whisper only (no OCR)?**
   - Would save 0-10% (OCR is parallel)
   - But lose critical FTC violation detection (disclosures in graphics)
   
3. **Cache transcripts?**
   - If auditing same video twice, reuse first transcript
   - Saves 100-200s on cache hit
   - Implementation: Hash video file, check DB before VI processing

4. **Async notification?**
   - Instead of polling every 30s, use VI webhooks
   - Saves polling overhead (~5-10s of request time)
   - Still can't speed up ML itself

**Real-World Latency Expectation:**
- For production: Accept 3-4 min E2E as baseline
- Communicate to users upfront: "Audit takes ~4 minutes"
- Make API async to hide this from user (return task_id immediately)

---

### Q8: How Do You Ensure Reproducibility?

**The Challenge:**
- Code changes → different output
- KB changes → different output
- LLM model updates → different output
- Video processing changes → different output

**Current Approach: None**
- No versioning of components
- No audit trail of "this video was audited with these versions"

**Required for Compliance:**
- Must be able to answer: "Why did we flag violation X on video Y?"
- Must be able to replay with same versions and get same result

**Solution: Audit Record Versioning**

`python
class AuditRecord(BaseModel):
    audit_id: str
    video_url: str
    timestamp: datetime
    
    # Version tracking
    vi_model_version: str  # "2024-07-01"
    transcript: str
    ocr_text: str
    
    kb_version: str  # "2024-07-15"
    kb_documents_used: List[str]  # Document IDs/hashes
    
    llm_model: str  # "gpt-4"
    llm_temperature: float  # 0.0
    system_prompt_version: str  # "2024-07-01"
    
    violations: List[Violation]
    
    # Reproducibility check
    reproducibility_hash: str  # hash of all inputs
`

**Implementation Steps:**

1. **Version the System Prompt**
   `python
   SYSTEM_PROMPT_V1 = """You are..."""  # Tag with date
   `

2. **Version KB Documents**
   `python
   KB_DOCUMENT = {
       "id": "FTC_GUIDE_V2024_07_15",
       "hash": "sha256:abc123",
       "version": "2024-07-15"
   }
   `

3. **Version VI Model** (auto-tracked by Azure)
   `python
   vi_response = vi_client.get_video_info(video_id)
   vi_model_version = vi_response.metadata.model_version
   `

4. **Store Full Audit Input**
   `python
   audit_record.transcript_hash = hash(transcript)
   audit_record.ocr_hash = hash(ocr)
   `

5. **Compute Reproducibility Hash**
   `python
   repro_hash = hash(
       transcript + ocr + kb_version + 
       llm_model + system_prompt_version
   )
   `

**On Re-Audit:**
- Pull same versions from storage
- Run workflow again
- Compare: new_hash == old_hash? ✓ Reproducible

**Test Case:**
`python
def test_reproducibility():
    v1_result = audit_with_versions(
        video_url="...",
        kb_version="2024-07-01",
        prompt_version="2024-07-01"
    )
    v1_result = audit_with_versions(
        video_url="...",  # Same video
        kb_version="2024-07-01",
        prompt_version="2024-07-01"
    )
    assert v1_result.violations == v2_result.violations
`

---

### Q9: Serverless Gaps—Why Is Azure Functions Configured but Empty?

**Current Setup:**
- /azure_functions/ directory exists
- Functions placeholder configured
- Never invoked

**Why This Matters:**

Video Indexer has webhook capability:
- After processing completes, VI calls your webhook
- Instead of polling every 30s (which wastes time + API quota)
- VI pushes notification to Function
- Function triggers workflow

**Expected Implementation:**

`python
# azure_functions/VideoProcessed/__init__.py
import azure.functions as func
from backend.src.graph.workflow import workflow

async def main(req: func.HttpRequest) -> func.HttpResponse:
    # VI webhook posts here when video processing complete
    video_id = req.json.get("video_id")
    
    # Trigger audit workflow
    result = await workflow.ainvoke({
        "video_id": video_id,
        "processing_complete": True
    })
    
    return func.HttpResponse(
        json.dumps({"status": "audit_started"}),
        status_code=202
    )
`

**Benefits of Webhooks:**
1. **Eliminates polling** (saves 100+ API calls)
2. **Real-time trigger** (VI processing done → audit starts immediately)
3. **Auto-scales** (Azure Functions scale per message in queue)
4. **Decouples** (API server doesn't need to poll)

**Current Gap:**
- Polling loop in VideoIndexerService.wait_for_processing() (lines 97-118)
- Polls every 30s for status
- Blocks during entire wait
- Wastes API quota

**To Fix:**
1. Register webhook URL with Azure VI account
2. Implement webhook handler (shown above)
3. Remove polling loop
4. Save ~100-200s per audit (move to event-driven)

---

### Q10: How Do You Manage Data Privacy?

**Current State: None**
- Transcripts stored in-memory
- OCR text stored in-memory
- No encryption
- No retention policy
- No GDPR/CCPA handling

**Privacy Risks:**

1. **Speaker Identification**
   - OCR can extract names from video overlays
   - Speech recognition identifies speakers
   - No consent tracking for PI collection

2. **Video Content**
   - If you store transcripts, you're storing derivative content
   - GDPR: "Users have right to be forgotten"
   - But transcript tied to video URL → can't delete

3. **Compliance Data**
   - Audit records might reference specific speakers/influencers
   - This is business data (legitimate), but needs access control

**Recommended Policy:**

`python
class DataRetentionPolicy:
    RETENTION_DAYS = 90  # After 90 days, auto-delete
    ANONYMIZATION = "REDACT_SPEAKERS"  # Remove speaker names
    ENCRYPTION = "AES-256-GCM"
    
    def apply_to_audit_record(record: AuditRecord):
        # Redact speaker names
        record.transcript = redact_speaker_names(record.transcript)
        
        # Hash video URL (can't identify original)
        record.video_url_hash = sha256(record.video_url)
        
        # Mark PII fields
        record.pii_redacted_at = datetime.now()
`

**GDPR Compliance:**
1. Store consent (user agreed to video audit)
2. Allow deletion requests (right to be forgotten)
3. Provide audit records on request (right to access)
4. Delete after retention period (data minimization)

**CCPA Compliance:**
1. Disclose data usage in privacy policy
2. Allow opt-out from tracking
3. Provide data download
4. Delete on request

---

### Q11: Observability & Monitoring Gaps

**Current Setup:**
- Azure Monitor integrated (basic)
- OpenTelemetry configured
- Tracks: HTTP requests, exceptions, response times

**Missing:**
- ❌ Custom business metrics
- ❌ Alerting
- ❌ SLO tracking
- ❌ Cost tracking
- ❌ Audit trail logging

**Recommended Metrics:**

`python
from azure.monitor.opentelemetry import AutoLoggingOptions

meter = AutoLoggingOptions.get_meter()

# Business Metrics
meter.create_counter(
    "audit_total",
    unit="1",
    description="Total audits run"
).add(1, {"status": "success", "video_length_minutes": 10})

meter.create_histogram(
    "audit_duration_seconds",
    unit="s",
    description="Time to complete audit"
).record(175, {"video_length_minutes": 10})

meter.create_gauge(
    "violations_found",
    unit="1",
    description="Violations per audit"
).record(3, {"severity": "HIGH"})

# Cost Metrics
meter.create_counter(
    "cost_usd",
    unit="USD",
    description="Cost per audit"
).add(0.50, {"service": "video_indexer"})

# Quality Metrics
meter.create_gauge(
    "false_positive_rate",
    unit="percent",
    description="% of violations that were incorrect"
).record(15, {})
`

**Recommended Alerts:**

| Alert | Threshold | Action |
|-------|-----------|--------|
| Error Rate | >5% | Page on-call |
| P99 Latency | >5 min | Investigate bottleneck |
| Cost/Audit | >.00 | Review KB size |
| False Positive | >20% | Retrain prompt |

---

### Q12: How Would You Scale to 1000s of Concurrent Audits?

**Current Architecture Limit:**
- Single FastAPI server
- ~50-100 concurrent requests (default worker threads)
- Each blocks for 3-4 minutes
- Throughput: ~0.5-1 audit/min (severely limited)

**For 1000 Concurrent Audits:**

**Tier 1: API Layer (Horizontal Scale)**
`
Load Balancer
├── FastAPI Server 1
├── FastAPI Server 2
├── FastAPI Server 3
└── FastAPI Server 4
`
- Each server: 100 workers
- Load balancer distributes requests
- Result: 400 concurrent async operations

**Tier 2: Job Queue (Decouple Submission from Processing)**
`
HTTP POST /audit → Queue → Return task_id immediately
Background Workers consume from queue
`
- Azure Queue Storage (or Service Bus)
- Priority queues (HIGH/MEDIUM/LOW)
- Max 10,000 jobs queued

**Tier 3: Worker Pool (Auto-Scale)**
`
Queue Trigger → Azure Functions (auto-scale)
OR
Queue Trigger → Kubernetes (manual scale)
`
- Auto-scales: 1 function when 1 job, 100 when queue depth=100
- Each function runs 1 workflow (no worker thread limit)
- Result: Unlimited throughput

**Tier 4: Database Layer (Persist State)**
`
AuditRecord table (PostgreSQL)
├── audit_id (PK)
├── status (queued/processing/done/error)
├── video_url
├── violations
└── created_at
`
- Track audit status
- Allow polling endpoint: GET /audit/{task_id}
- Replay on crash

**Tier 5: Caching Layer (Reduce Latency)**
`
Redis Cache
├── video_url → transcript (TTL: 24h)
├── query_text → kb_search_results (TTL: 7d)
└── model_predictions → violations (TTL: never)
`
- Cache transcripts (reused videos)
- Cache KB searches (common queries)
- Cache model predictions (same input = same output at temp=0.0)

**Bottleneck at Scale: Azure Video Indexer**
- VI has per-account concurrency limit (~100-200 parallel uploads)
- After that, VI jobs queue
- Solution: Distribute across multiple VI accounts (in different regions if needed)

**Architecture Diagram:**
`
User Requests
    ↓
[Load Balancer]
    ↓
├─ FastAPI 1 ──┐
├─ FastAPI 2 ──┼─→ [Redis Cache] ──┐
├─ FastAPI 3 ──┤                    ├─→ [Azure Queue Storage]
├─ FastAPI 4 ──┘                    │
                                    ├─→ [PostgreSQL]
                                    │
                                    ├─→ [Azure Functions]
                                    │   ├─ Worker 1
                                    │   ├─ Worker 2
                                    │   ├─ ... auto-scale
                                    │
                                    └─→ [Azure Video Indexer]
`

**Expected Performance:**
- P99 Latency: 4 min (VI processing is fundamental limit)
- Throughput: 1000+ audits/hour
- Cost: \-500 per 1000 (unchanged per-audit cost)
- Ops Cost: \-2000/month for infrastructure

---

## PRODUCTION-READINESS MATRIX

| Category | Current | Target | Effort | Priority |
|----------|---------|--------|--------|----------|
| Async Processing | ❌ | ✅ | 2 days | P0 |
| State Persistence | ❌ | ✅ | 3 days | P0 |
| Retry Logic | ❌ | ✅ | 1 day | P0 |
| Input Validation | ⚠️ | ✅ | 1 day | P0 |
| Error Handling | ⚠️ | ✅ | 2 days | P1 |
| Audit Logging | ❌ | ✅ | 2 days | P1 |
| Secret Management | ❌ (env vars) | ✅ (Key Vault) | 1 day | P1 |
| Cost Tracking | ❌ | ✅ | 1 day | P1 |
| Health Checks | ❌ | ✅ | 0.5 day | P2 |
| Test Coverage | ❌ | 80%+ | 5 days | P2 |
| Monitoring | ⚠️ (basic) | ✅ (comprehensive) | 3 days | P2 |
| Documentation | ⚠️ (code only) | ✅ (runbooks) | 2 days | P3 |

**Total Effort to Production:** ~3-4 weeks (1-2 engineers)

---

## SUMMARY

**Brand Guardian is well-designed but needs:**

1. **Async API** (biggest blocker)
2. **Persistence layer** (for audit trail)
3. **Retry + error handling**
4. **Security hardening**
5. **Observability**

After these fixes, it's enterprise-grade and ready to scale.
