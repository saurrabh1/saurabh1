# Jarvis — Technical Roadmap
## Phase 2 MVP Build Sequence

---

## Milestone Breakdown

```
PHASE 2 MVP — 8 Sprints (2 weeks each)
────────────────────────────────────────────────────────────

Sprint 1-2:  Foundation & Wake Word
Sprint 3-4:  Agent Manager Core + Neural Dot
Sprint 5:    Web Explorer + Workspace Sync Sub-Agents
Sprint 6:    Notification Daemon
Sprint 7:    Dashboard UI (Scrollable Memory)
Sprint 8:    Integration, Polish, and Latency Tuning
```

---

## Sprint 1-2: Foundation & Wake Word (Weeks 1-4)

### Deliverables
- [ ] Supabase project created, schema migrations run (`02_DATABASE_SCHEMA.md`)
- [ ] Android Foreground Service with Porcupine SDK running
- [ ] Custom "Hey Jarvis" wake word model downloaded and bundled
- [ ] Deepgram WebSocket streaming STT connected post-wake
- [ ] FastAPI skeleton deployed on Railway with WebSocket endpoint
- [ ] Basic TTS response played back through phone speaker (earbuds next sprint)

### Key Technical Risks
- Battery optimization whitelist UX — must surface clearly in onboarding
- Deepgram WebSocket reconnect on network drop

---

## Sprint 3-4: Agent Manager Core + Neural Dot (Weeks 5-8)

### Deliverables
- [ ] LangGraph graph wired with all nodes (Intent Classifier → Router → Synthesizer)
- [ ] Claude claude-sonnet-4-6 intent classification with structured output working
- [ ] Neural Dot Categorizer Agent writing to all 4 bucket tables
- [ ] Embedding generation (text-embedding-3-small) on every bucket write
- [ ] Scheduled notification rows created when reminders have `due_at`
- [ ] Simple `conversation` intent handled (no sub-agent, direct Claude response)

---

## Sprint 5: Sub-Agents (Weeks 9-10)

### Deliverables
- [ ] Web Explorer agent with Tavily API integration, voice-friendly response trimming
- [ ] Workspace Sync agent with at least one integration (Google Drive recommended)
- [ ] Response synthesizer enforces TTS length limit (≤ 60 words for earbuds)

---

## Sprint 6: Notification Daemon (Weeks 11-12)

### Deliverables
- [ ] APScheduler daemon running as background thread in FastAPI
- [ ] FCM push notification integration (Android)
- [ ] TTS message generated and pushed exactly 5 minutes before `due_at`
- [ ] `is_fired` flag set atomically to prevent duplicate sends

---

## Sprint 7: Dashboard UI (Weeks 13-14)

### Deliverables
- [ ] Next.js app deployed on Vercel, authenticated via Supabase Auth
- [ ] 4 bucket tabs: Personal / Reminders / Knowledge / Projects
- [ ] Chronological scrollable list per bucket (summary + timestamp + tags)
- [ ] Semantic search bar (vector search via Supabase `match_` function)
- [ ] Mark reminder as complete from UI

---

## Sprint 8: Integration & Polish (Weeks 15-16)

### Deliverables
- [ ] End-to-end latency measured and tuned (target: < 4s for simple queries)
- [ ] Bluetooth earbud mic routing tested (HFP profile)
- [ ] Error recovery: STT failure → graceful "I didn't catch that" TTS response
- [ ] False positive wake word tuning (sensitivity calibrated per user environment)
- [ ] Rate limits and cost monitoring on all API calls

---

## Cost Estimate (Personal Project, ~100 interactions/day)

| Service | Usage/month | Est. Cost/month |
|---------|------------|----------------|
| Supabase (free tier) | Up to 500MB DB | $0 |
| Upstash Redis (free tier) | Up to 10K req/day | $0 |
| Railway (Hobby) | 1 FastAPI instance | $5 |
| Vercel (free) | Next.js dashboard | $0 |
| Picovoice Porcupine | 1 wake word, free tier | $0 |
| Deepgram (STT) | ~100 min/day = 3000 min | ~$12 |
| Anthropic Claude | ~100 intent calls + agents | ~$8 |
| OpenAI TTS | ~100 responses × 100 tokens | ~$3 |
| Tavily API (free tier) | 1000 searches/month | $0 |
| **Total** | | **~$28/month** |

---

## Repository Structure (Target)

```
jarvis/
├── backend/
│   ├── main.py                    # FastAPI app, WebSocket handler
│   ├── agent_manager/
│   │   ├── graph.py               # LangGraph definition
│   │   ├── intent_classifier.py   # Claude classification node
│   │   ├── router.py              # Conditional edge dispatch
│   │   └── agents/
│   │       ├── web_explorer.py
│   │       ├── workspace_sync.py
│   │       ├── neural_dot.py
│   │       └── notification_scheduler.py
│   ├── memory/
│   │   ├── database.py            # Supabase client
│   │   └── embeddings.py          # OpenAI embedding wrapper
│   ├── daemon/
│   │   └── notification_daemon.py # APScheduler background job
│   └── tts/
│       └── synthesizer.py         # TTS provider abstraction
│
├── mobile/                        # Flutter Android app
│   ├── lib/
│   │   ├── services/
│   │   │   ├── wake_service.dart  # Porcupine + AudioRecord
│   │   │   └── stt_stream.dart    # Deepgram WebSocket
│   │   └── main.dart
│   └── android/
│       └── src/main/
│           └── JarvisWakeService.kt
│
├── dashboard/                     # Next.js UI
│   ├── app/
│   │   ├── page.tsx               # Bucket tabs + scroll view
│   │   └── api/
│   └── components/
│       ├── BucketView.tsx
│       └── MemoryCard.tsx
│
└── docs/
    └── jarvis/                    # This directory
        ├── 01_SYSTEM_ARCHITECTURE.md
        ├── 02_DATABASE_SCHEMA.md
        ├── 03_WAKE_WORD_SPEC.md
        └── 04_TECHNICAL_ROADMAP.md
```
