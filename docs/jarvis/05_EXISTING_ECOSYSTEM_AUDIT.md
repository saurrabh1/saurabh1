# Jarvis — Existing Ecosystem Audit
## "Don't Build From Scratch" — Framework Selection & Adaptation Plan

---

## Guiding Principle

Before writing a single line of custom code, audit what already exists.
The goal: identify the highest-leverage existing open-source framework that
covers the most ground, then layer our specific Jarvis requirements on top.

---

## Top Candidate Frameworks

### 1. Pipecat (pipecat-ai/pipecat) ⭐ RECOMMENDED BASE

**Repo:** `pipecat-ai/pipecat`
**Stars:** ~7,000+ (as of mid-2025)
**Language:** Python
**License:** BSD-2-Clause (permissive)
**Maintained by:** Daily.co (actively maintained, production-grade)

**What it already does:**
- Full real-time STT → LLM → TTS pipeline out of the box
- Frame-based event bus architecture (audio frames, text frames, LLM frames)
- Native adapters for: Deepgram, OpenAI Whisper, AssemblyAI (STT)
- Native adapters for: OpenAI, Anthropic Claude, Gemini (LLM)
- Native adapters for: ElevenLabs, OpenAI TTS, Cartesia (TTS)
- VAD (Voice Activity Detection) built-in via Silero VAD
- Interruption handling (user can cut off Jarvis mid-sentence)
- WebRTC transport (Daily.co) and WebSocket transport
- Works on macOS and Linux

**What it does NOT do (we build this):**
- Wake word detection (add Porcupine before the pipeline)
- Multi-agent routing / intent classification (add LangGraph on top)
- Persistent memory / Neural Dot buckets (add Supabase layer)
- Proactive notification daemon (add APScheduler sidecar)
- iOS mobile companion (build separately)

**Assessment: 70% of the voice pipeline is already built. Use this.**

---

### 2. LiveKit Agents (livekit/agents)

**Repo:** `livekit/agents`
**Stars:** ~3,000+ (as of mid-2025)
**Language:** Python
**License:** Apache 2.0
**Maintained by:** LiveKit (YC-backed, production-grade)

**What it does:**
- Real-time voice agents via WebRTC rooms
- STT → LLM → TTS pipeline similar to Pipecat
- Better for multi-user / multi-room scenarios
- Has VoicePipelineAgent and MultimodalAgent abstractions
- Good for cloud-hosted deployments

**Why NOT first choice for Jarvis:**
- Optimized for multi-user call infrastructure (not personal single-user)
- Requires LiveKit cloud or self-hosted LiveKit server (extra infra)
- Overkill for a personal always-on assistant
- Less flexible for custom pipeline hacking

---

### 3. mem0 (mem0ai/mem0)

**Repo:** `mem0ai/mem0`
**Stars:** ~20,000+ (as of mid-2025)
**Language:** Python
**License:** Apache 2.0

**What it does:**
- Memory layer for AI agents — exactly what Neural Dot needs
- Extracts facts, preferences, and memories from conversation
- Stores in vector DB (supports Qdrant, Pinecone, Chroma, pgvector)
- Retrieves relevant memories when context is needed
- Has `add()`, `search()`, `get_all()` API

**Why relevant:**
- Instead of building the Neural Dot categorization from scratch,
  use mem0 as the memory extraction layer, then route to our isolated buckets.

**Limitation:** Does not enforce strict bucket isolation by itself.
We wrap it: mem0 extracts the memory, our classifier puts it in the right bucket table.

---

### 4. NVIDIA voice-agent-examples (NVIDIA/voice-agent-examples)

**Repo:** `NVIDIA/voice-agent-examples`
**Stars:** ~54 (as of May 2026)
**Language:** Python
**Based on:** Pipecat

**What it does:**
- Multi-agent orchestration demos built on Pipecat
- Shows how to plug a router / dispatcher into the pipeline
- Good reference architecture for how NVIDIA implements sub-agent dispatch

**Use as:** Reference implementation only. Our codebase will be cleaner.

---

### 5. OpenWakeWord (dscripka/openWakeWord)

**Repo:** `dscripka/openWakeWord`
**Stars:** ~4,000+ (as of mid-2025)
**Language:** Python
**License:** Apache 2.0

**What it does:**
- On-device wake word detection in Python
- Runs on CPU, ~3-5% CPU usage
- Pre-trained models: "hey jarvis", "alexa", "hey siri" (community-trained)
- No cloud calls, fully local

**Why this matters:**
- There is ALREADY a community-trained "hey_jarvis" model available for OpenWakeWord
- On macOS daemon, this means: zero custom training required
- On iPhone: does not run (Python-only), so Porcupine iOS SDK still needed for mobile

---

## Decision Matrix

| Component | Build From Scratch | Use Existing | What to Use |
|-----------|-------------------|--------------|-------------|
| Voice pipeline (STT→LLM→TTS) | ❌ Too complex | ✅ | **Pipecat** |
| Wake word (macOS daemon) | ❌ | ✅ | **OpenWakeWord** (free, no training) |
| Wake word (iPhone) | ❌ | ✅ | **Porcupine iOS SDK** |
| Intent routing | Partially | ✅ | **LangGraph** on top of Pipecat |
| Memory extraction | ❌ | ✅ | **mem0** for extraction |
| Bucket storage (Neural Dot) | ✅ Custom | — | **Supabase** schema (our design) |
| Web search | ❌ | ✅ | **Tavily** API |
| Notification daemon | ✅ Custom | — | APScheduler sidecar |
| iPhone app | ✅ Custom | — | Swift (native) or React Native |

---

## Recommended Stack (Adapt-First Approach)

```
Layer 1: Wake Word Detection
  macOS:   OpenWakeWord (Python, free, "hey jarvis" model exists)
  iPhone:  Porcupine iOS SDK (free tier)

Layer 2: Voice Pipeline
  Pipecat (pipecat-ai/pipecat)
    - Transport: WebSocket (phone) or local mic (Mac)
    - STT: Deepgram Nova-2
    - LLM: Claude claude-sonnet-4-6 via Anthropic
    - TTS: OpenAI TTS tts-1-hd

Layer 3: Agent Router (our custom layer on Pipecat)
  LangGraph graph injected as Pipecat LLM frame processor
    - Intent Classifier → Router → Sub-agents
    - Sub-agents call tools: Tavily, GitHub API, local FS

Layer 4: Memory
  mem0 (extracts facts from utterances)
  + Custom bucket classifier (routes to correct Supabase table)
  + Supabase PostgreSQL + pgvector (isolated bucket storage)

Layer 5: Proactive Notifications
  APScheduler daemon (runs alongside Pipecat server)
  + Apple Push Notification Service (APNs) → iPhone
  + Local notification on Mac via osascript / terminal-notifier
```

---

## Pipecat Integration Point for the Agent Router

The cleanest integration: replace Pipecat's default LLM processor with our
LangGraph-powered router. Pipecat's frame bus handles the audio pipeline;
LangGraph handles the intelligence.

```python
# How Pipecat normally works (simple):
pipeline = Pipeline([
    transport.input(),
    stt,
    llm,          # ← replace this with our LangGraph router
    tts,
    transport.output()
])

# How Jarvis works (our customization):
pipeline = Pipeline([
    transport.input(),
    stt,
    jarvis_router,   # ← our LangGraph agent manager as a frame processor
    tts,
    transport.output()
])

# jarvis_router intercepts TextFrame from STT,
# runs full LangGraph graph, returns response TextFrame to TTS
```

---

## First Action Items (Before Writing Code)

1. **Clone pipecat-ai/pipecat** → run the basic voice example locally on Mac
2. **Install openWakeWord** → test "hey jarvis" model — does it trigger accurately?
3. **Clone mem0ai/mem0** → test memory extraction on sample utterances
4. **Audit NVIDIA/voice-agent-examples** → understand their multi-agent Pipecat pattern
5. **Then** start building Jarvis custom layer on top of the validated stack

This order prevents the classic trap: building infrastructure that an
existing library already handles better.

---

## Repos to Star / Fork (Your Action)

| Repo | URL | Action |
|------|-----|--------|
| pipecat | `github.com/pipecat-ai/pipecat` | Fork + local clone |
| openWakeWord | `github.com/dscripka/openWakeWord` | Clone + test |
| mem0 | `github.com/mem0ai/mem0` | pip install, test |
| NVIDIA examples | `github.com/NVIDIA/voice-agent-examples` | Read + reference |
| Porcupine | `github.com/Picovoice/porcupine` | SDK only (pip install) |
