# Jarvis — Multi-Agent Voice Ecosystem
## System Architecture & Technical Roadmap (MVP Phase 2)

---

## 1. High-Level System Map

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                    │
│                                                                         │
│  [Earbuds / BT Audio]  ◄──TTS Audio──  [Mobile App (Flutter/Android)]  │
│                                              │                          │
│                          Wake Word Engine    │  Mic Stream (16kHz PCM) │
│                          (Porcupine DSP) ────┘                          │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │  WebSocket (audio stream)
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         BACKEND CORE (FastAPI)                           │
│                                                                         │
│  ┌──────────────┐    ┌─────────────────────────────────────────────┐   │
│  │  STT Engine  │    │           AGENT MANAGER (LangGraph)          │   │
│  │  (Deepgram   │───►│                                             │   │
│  │   Streaming) │    │  ┌─────────────┐    ┌──────────────────┐   │   │
│  └──────────────┘    │  │  Intent     │    │  Router Node     │   │   │
│                       │  │  Classifier │───►│  (Conditional    │   │   │
│  ┌──────────────┐    │  │  (Claude)   │    │   Edge Dispatch) │   │   │
│  │  TTS Engine  │    │  └─────────────┘    └──────┬───────────┘   │   │
│  │  (ElevenLabs │◄───│                            │               │   │
│  │   / OAI TTS) │    │         ┌──────────────────┴──────────┐    │   │
│  └──────────────┘    │         ▼          ▼         ▼        ▼    │   │
│                       │  [Web Explorer] [Workspace] [NeuralDot] [Notif]│
│  ┌──────────────┐    │  Sub-Agent A    Sub-Agent B  Categorizer Sched. │
│  │  Notif.      │◄───│                                             │   │
│  │  Daemon      │    └─────────────────────────────────────────────┘   │
│  │  (APScheduler│                                                       │
│  └──────────────┘                                                       │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                             ▼
        ┌──────────────────────┐    ┌──────────────────────┐
        │  PostgreSQL + pgvec  │    │  Redis (Upstash)     │
        │  (Supabase)          │    │  Session ctx + PubSub│
        │  Neural Dot Buckets  │    │  Notification queue  │
        └──────────────────────┘    └──────────────────────┘
```

---

## 2. Agent Manager — Intelligent Router Design

### 2.1 Core Design Principle

The Agent Manager is a **stateful LangGraph graph** where every node is a discrete processing unit and edges are conditional branches based on classified intent. The LLM never routes blindly — it emits a structured JSON routing decision, and the graph enforces the path.

### 2.2 Intent Classification Schema

Every incoming transcribed utterance passes through the Intent Classifier node first. It calls Claude with a structured output tool:

```python
class IntentDecision(BaseModel):
    intent: Literal[
        "web_search",        # → Sub-Agent A (Web Explorer)
        "workspace_sync",    # → Sub-Agent B (Cloud/Workspace)
        "neural_dot",        # → Neural Dot Categorizer
        "schedule_notify",   # → Notification Scheduler
        "conversation",      # → Direct LLM response (no sub-agent)
    ]
    confidence: float        # 0.0 – 1.0
    bucket: Optional[Literal[
        "personal",
        "reminders",
        "knowledge",
        "projects",
    ]]                       # Required only when intent == "neural_dot"
    params: dict             # Route-specific parameters
    raw_utterance: str       # Original transcribed text
```

**Example classification outputs:**

| User says | Intent | Bucket | params |
|-----------|--------|--------|--------|
| "What's the latest on GPT-5?" | `web_search` | — | `{"query": "GPT-5 latest news"}` |
| "Pull up my notes from the Jarvis project" | `workspace_sync` | — | `{"resource": "notes", "filter": "Jarvis project"}` |
| "Note this down: I think the pricing model should shift to usage-based" | `neural_dot` | `projects` | `{"content": "pricing model → usage-based"}` |
| "Remind me about the 10 AM meeting" | `neural_dot` + `schedule_notify` | `reminders` | `{"due_at": "10:00", "task": "10 AM meeting"}` |
| "Hey, how are you?" | `conversation` | — | `{}` |

### 2.3 LangGraph State Machine

```python
# State passed between all nodes
class JarvisState(TypedDict):
    session_id: str
    user_id: str
    raw_utterance: str
    intent: IntentDecision
    agent_response: str
    memory_written: bool
    tts_payload: str

# Graph definition
graph = StateGraph(JarvisState)

graph.add_node("intent_classifier",  intent_classifier_node)
graph.add_node("web_explorer",        web_explorer_agent)
graph.add_node("workspace_sync",      workspace_sync_agent)
graph.add_node("neural_dot",          neural_dot_agent)
graph.add_node("notify_scheduler",    notification_scheduler_node)
graph.add_node("direct_convo",        direct_conversation_node)
graph.add_node("response_synthesizer", response_synthesizer_node)

graph.set_entry_point("intent_classifier")

# Conditional routing — the router edge
graph.add_conditional_edges(
    "intent_classifier",
    route_intent,   # pure function: state → node name
    {
        "web_search":      "web_explorer",
        "workspace_sync":  "workspace_sync",
        "neural_dot":      "neural_dot",
        "schedule_notify": "notify_scheduler",
        "conversation":    "direct_convo",
    }
)

# All agents converge back to synthesizer
for node in ["web_explorer", "workspace_sync", "neural_dot",
             "notify_scheduler", "direct_convo"]:
    graph.add_edge(node, "response_synthesizer")

graph.add_edge("response_synthesizer", END)
```

### 2.4 Sub-Agent Designs

#### Sub-Agent A — Web Explorer

```
Tools available:
  - tavily_search(query, max_results=5)       → real-time web results
  - perplexity_query(query)                    → synthesized answer
  - scrape_url(url)                            → full page content

Flow:
  1. Receive web_search intent + query param
  2. Run tavily_search → get top 5 results
  3. Synthesize with Claude: "Given these results, answer: {query}"
  4. Return concise voice-friendly response (≤ 3 sentences for TTS)
```

#### Sub-Agent B — Cloud / Workspace Sync

```
Tools available:
  - google_drive_search(query, file_type)
  - notion_query(database_id, filter)
  - github_search_code(repo, query)
  - read_local_file(path)                      → sandboxed local FS

Flow:
  1. Receive workspace_sync intent + resource params
  2. Fan out to relevant API tools in parallel
  3. Consolidate results → summarize with Claude
  4. Return context-rich response
```

#### Sub-Agent C — Neural Dot Categorizer

```
Flow:
  1. Receive neural_dot intent + pre-classified bucket
  2. Extract structured entry: { summary, tags, due_at? }
  3. Generate embedding (text-embedding-3-small)
  4. Write to isolated bucket table in PostgreSQL
  5. If bucket == "reminders" AND due_at present → also write to
     scheduled_notifications table
  6. Confirm: "Got it, I've saved that to your [bucket] notes."
```

#### Notification Daemon (Background)

```
Runs as APScheduler job every 60 seconds:
  SELECT * FROM scheduled_notifications
  WHERE is_fired = FALSE
    AND (scheduled_for - INTERVAL '5 minutes') <= NOW();

For each result:
  1. Build TTS message: "Hey Saurabh, [title] is happening in 5 minutes."
  2. Push via FCM (Firebase) to user's device
  3. Device plays TTS audio directly to earbuds
  4. Mark is_fired = TRUE
```

---

## 3. Tech Stack Decisions

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Mobile Client | Flutter (Android first) | Best audio API access, cross-platform |
| Wake Word | Picovoice Porcupine SDK | On-device, <1% CPU, custom "Hey Jarvis" |
| STT | Deepgram Nova-2 (streaming) | Lowest latency real-time transcription |
| LLM / Brain | Claude claude-sonnet-4-6 (Anthropic) | Best instruction-following for routing |
| Agent Framework | LangGraph | State machine for multi-agent orchestration |
| TTS | OpenAI TTS (tts-1-hd) | Low latency, natural voice |
| Backend | Python FastAPI + uvicorn | Async WebSocket + REST |
| Web Search | Tavily API | Purpose-built for LLM agents, real-time |
| Database | Supabase (PostgreSQL + pgvector) | Managed, free tier, vector search |
| Cache / Queue | Upstash Redis | Serverless Redis, free tier, pub/sub |
| Push Notif. | Firebase FCM | Reliable Android push, triggers audio |
| Hosting | Railway.app | One-click Python deploy, free tier |
| Dashboard UI | Next.js (React) | Fast SSR, easy Supabase integration |

---

## 4. End-to-End Voice Request Flow

```
1. User says "Hey Jarvis, search for the latest AI models benchmark"

2. [Porcupine] detects wake word on-device (screen off, <1% CPU)
   → activates microphone

3. [Deepgram WebSocket] begins streaming PCM audio → returns text tokens
   → "search for the latest AI models benchmark"

4. [FastAPI WS handler] receives full utterance → pushes to Agent Manager

5. [Intent Classifier Node]
   → calls Claude with structured tool
   → returns: { intent: "web_search", params: { query: "AI models benchmark 2025" } }

6. [Router] dispatches to Web Explorer Agent

7. [Web Explorer]
   → tavily_search("AI models benchmark 2025")
   → gets 5 results → synthesizes with Claude
   → returns: "Based on recent benchmarks, Claude claude-sonnet-4-6 leads in reasoning..."

8. [Response Synthesizer] trims for TTS length, adds persona

9. [ElevenLabs / OAI TTS] converts text → audio stream

10. Audio plays through Bluetooth earbuds
    Total latency target: < 4 seconds end-to-end
```

---

## 5. Clarifying Questions for Saurabh

Before Phase 2 implementation begins, I need your input on:

1. **Platform first?** Android only for MVP, or iOS too? (Porcupine supports both but build complexity doubles)
2. **Earbud target?** AirPods (iOS), Galaxy Buds (Android), or agnostic? This affects the BT audio routing strategy.
3. **Cloud accounts to integrate?** Which of these for Workspace Sync: Google Drive, Notion, GitHub, or all three?
4. **Self-hosted vs managed?** Are you comfortable with a $0/month stack (Supabase + Railway free tiers) or do you want to own the infra?
5. **Voice persona?** Should Jarvis have a specific ElevenLabs voice, or is a standard neural TTS voice fine for MVP?
6. **Dashboard UI priority?** Is the web dashboard a Phase 2 deliverable or can it slip to Phase 3 (voice-only MVP first)?
