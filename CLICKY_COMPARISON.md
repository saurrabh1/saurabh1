# CLICKY WINDOWS vs CLICKY macOS - COMPLETE CODE-LEVEL COMPARISON

## **1. CORE ARCHITECTURE**

| Aspect | macOS (farzaa/clicky) | Windows (Bitshank-2338/clicky-windows) |
|--------|----------------------|---------------------------------------| 
| Language | Swift | Python 3.11+ |
| UI Framework | SwiftUI + NSPanel | PyQt6 |
| Entry Point | leanring_buddyApp.swift | main.py |
| State Machine | CompanionManager.swift (Observable) | companion_manager.py (QObject + async) |
| Threading | Combine publishers | asyncio + threading |
| Config File | UserDefaults (plist) | .env file |
| Analytics | PostHog SDK built-in | None |

---

## **2. VOICE INPUT PIPELINE**

| Stage | macOS | Windows |
|-------|-------|---------|
| Hotkey Monitor | GlobalPushToTalkShortcutMonitor.swift | hotkey.py (keyboard library) |
| Audio Input | AVCaptureDevice | sounddevice library |
| Wake Word | Built-in detection | ambient_listener.py with detection |
| STT Providers | AssemblyAI, OpenAI, Apple Speech | Deepgram, OpenAI, whisper.cpp, Faster-Whisper |
| STT Code File | AssemblyAIStreamingTranscriptionProvider.swift | audio/stt/ folder (modular) |

---

## **3. SCREEN CAPTURE**

| Feature | macOS | Windows |
|---------|-------|---------|
| Capture Library | ScreenCaptureKit (macOS only) | mss (cross-platform) |
| Output Format | JPEG + pixel dimensions | JPEG + base64 encoded |
| Multi-Monitor Support | Yes (captureAllScreensAsJPEG) | Yes (capture_all_screens) |
| DPI Handling | Automatic scaling | Manual dpi_scale per screenshot |

---

## **4. LLM PROVIDERS (The Brain)**

| Feature | macOS | Windows |
|---------|-------|---------|
| Number of Providers | **1 (Claude only)** | **5 (Claude, OpenAI, Gemini, Copilot, Ollama)** |
| Provider Code | ClaudeAPI.swift | ai/ folder with separate files |
| Base Class | None (direct API) | BaseLLMProvider (abstract class) |
| Streaming | analyzeImageStreaming() | stream_response() (async generator) |
| Conversation History | In-memory only | SQLite database + in-memory per-app |
| History Limit | 10 exchanges | 20 exchanges per app |
| Runtime Switching | **No** | **Yes (via tray menu)** |
| Free Option | No | Yes (Ollama local) |

---

## **5. ELEMENT POINTING**

| Aspect | macOS | Windows |
|--------|-------|---------|
| Primary Method | Claude Computer Use (Anthropic API) | Hybrid: UIA tree + OCR + LLM grid |
| Fallback | None (requires Claude) | Universal grid locator (any LLM) |
| Accuracy | ~5px (pixel-perfect) | ~5px (UIA) or ~25-50px (grid) |
| Code Files | ElementLocationDetector.swift | ai/hybrid_pointer.py, ai/element_locator.py, ai/universal_locator.py |
| Coordinate Tags | [POINT:x,y:label:screenN] | [POINT:x,y:label:screen1] + whiteboard tags |

---

## **6. ANIMATION & OVERLAY**

| Feature | macOS | Windows |
|---------|-------|---------|
| Buddy Shape | Blue triangle (SwiftUI Path) | Blue triangle (Qt painter) |
| Cursor Following | Spring physics + 60fps timer | Qt event loop |
| Flight Path | Bezier arc animation | Bezier arc animation |
| Dwell Effect | Pulsing circle + speech bubble | Pulsing highlight + label |
| UI Code File | OverlayWindow.swift (~600 LOC) | ui/overlay.py (Qt) |
| Slow Mode Teacher | No | Yes (1.7x slower) |

---

## **7. TEXT-TO-SPEECH**

| Feature | macOS | Windows |
|---------|-------|---------|
| Providers | ElevenLabs + macOS system TTS | ElevenLabs, OpenAI, Edge TTS |
| Voice Switching | No | **Yes (runtime via voice command)** |
| Multilingual | No | **Yes (auto-detect + voice swap)** |
| Edge TTS Voices | N/A | 400+ voices available |

---

## **8. KNOWLEDGE JOURNAL & LEARNING**

| Feature | macOS | Windows |
|---------|-------|---------|
| Storage | None | SQLite (persistent) |
| Q&A Logging | No | **Every Q&A logged** |
| Spaced Repetition | No | **Yes (SM-2 algorithm)** |
| Review Intervals | N/A | 1, 3, 7, 14, 30, 60, 120 days |
| Journal Queries | N/A | "what did I cover today?" |
| Quiz Mode | No | **Yes (review flashcards)** |
| Code File | N/A | tutor_features/journal.py |

---

## **9. ADVANCED WINDOWS-ONLY FEATURES**

| Feature | macOS | Windows | Purpose |
|---------|-------|--------|---------|
| OCR Fallback | ❌ | ✅ | Read fine print using Tesseract |
| Code Mode | ❌ | ✅ | Auto-detect IDE + specialized prompts |
| Multilingual Auto | ❌ | ✅ | Detect language + switch voice |
| Lesson Recording | ❌ | ✅ | MP4 video @ 8fps + transcript |
| Workflow Capture | ❌ | ✅ | Record clicks/keys, ask "what did I do?" |
| Whiteboard Annotations | ❌ | ✅ | Draw arrows, circles, underlines on screen |
| Skills System | ❌ | ✅ | User-extensible voice triggers |
| Document Context | ❌ | ✅ | Drag-drop PDF/DOCX as context |
| Per-App Memory | ❌ | ✅ | Separate history per window |

---

## **10. SYSTEM INTEGRATION**

| Aspect | macOS | Windows |
|--------|-------|---------|
| Menu/Tray | NSStatusItem (menu bar) | Qt QSystemTrayIcon (tray) |
| Notifications | Panel only | Toast notifications |
| Auto-Start | SMAppService (login items) | Registry entry |
| Sleep/Wake | System-handled | Custom watchdog thread |
| Window Detection | Accessibility API | Win32 GetWindowText() |
| Diagnostics Export | None | Tray menu → text report |

---

## **11. CONFIGURATION & API KEYS**

| Item | macOS | Windows |
|------|-------|---------|
| API Keys Location | Cloudflare Worker (server) | .env file (local or env vars) |
| Ollama Support | Not supported | Full support (vision + text models) |
| GitHub Copilot | Not supported | Device-flow OAuth + token cache |
| Model Caching | N/A | 30-day TTL |
| Hotkey Combo | Ctrl + Option | Ctrl + Alt + Space |

---

## **12. PERMISSIONS & FIRST-RUN**

| Permission | macOS | Windows |
|------------|-------|---------|
| Microphone | AVCaptureDevice.requestAccess() | Implicit (requested once) |
| Screen Recording | ScreenCaptureKit + popup | None needed |
| Accessibility | AXIsProcessTrusted() required | Win32 API (implicit) |
| Permission Polling | 1.5s refresh cycle | Not needed |
| First-Time Flow | Onboarding video + email form | Setup wizard (Ollama + models) |

---

## **13. CODE SIZE & COMPLEXITY**

| Metric | macOS | Windows |
|--------|-------|---------|
| Main Logic | CompanionManager ~1,026 LOC | companion_manager.py ~1,175 LOC |
| UI Code % | ~60% of codebase (SwiftUI) | Modular (~400 LOC) |
| Provider Files | 1 (Claude only) | 5 (separate files) |
| Tutor Features | 0 (core only) | 10 subsystems |
| Total Modules | ~20 Swift files | ~40 Python files |

---

## **14. WHAT'S EXACTLY THE SAME ✅ (100% Identical)**

✅ Blue triangle cursor buddy  
✅ Push-to-talk hotkey activation  
✅ Screenshot-based vision AI  
✅ Element pointing with coordinates  
✅ Conversation history tracking  
✅ TTS audio playback  
✅ [POINT:x,y:label] tag format  
✅ Onboarding video/flow  
✅ Per-app context awareness  
✅ Streaming LLM responses  

---

## **15. WHAT'S DIFFERENT**

### **macOS = SIMPLER & POLISHED**
- ❌ No knowledge journal
- ❌ No spaced repetition
- ❌ Only Claude (no provider switching)
- ❌ No OCR fallback
- ❌ No multilingual support
- ✅ Simpler, more focused
- ✅ Native macOS performance
- ✅ Polished UI

### **Windows = FEATURE-RICH & EXTENSIBLE**
- ✅ 5 LLM providers (runtime switching)
- ✅ Knowledge journal with SM-2 spaced repetition
- ✅ OCR fallback for fine print
- ✅ Auto-detect language + multilingual responses
- ✅ Lesson recording (MP4 + transcript)
- ✅ Workflow capture (clicks/keys)
- ✅ Whiteboard annotations
- ✅ User-extensible skills system
- ✅ Document context (PDF drag-drop)
- ✅ Per-app conversation isolation

---

## **16. SIDE-BY-SIDE COMPARISON (QUICK VIEW)**

| Category | macOS | Windows | Winner |
|----------|-------|---------|--------|
| Simplicity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | macOS |
| Features | ⭐⭐ | ⭐⭐⭐⭐⭐ | Windows |
| Performance | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | macOS |
| Extensibility | ❌ | ⭐⭐⭐⭐⭐ | Windows |
| Learning Tools | ⭐ | ⭐⭐⭐⭐⭐ | Windows |
| Polish/UX | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | macOS |
| Cost | Free | Free | Tie |
| Setup Ease | ⭐⭐⭐ | ⭐⭐ | macOS |

---

## **17. FINAL SUMMARY**

| Aspect | Description |
|--------|-------------|
| **Same Core** | Both have blue triangle cursor buddy + vision-based pointing + conversational AI |
| **macOS Philosophy** | Polished, minimal, production-ready tutor for Mac users |
| **Windows Philosophy** | Research-grade learning platform with extensibility + advanced features |
| **Provider Strategy** | macOS locked to Claude; Windows supports 5 vendors with runtime switching |
| **Learning Features** | macOS minimal; Windows full journal + spaced repetition + quizzing |
| **Extensibility** | macOS closed; Windows open with skills system |
| **Best For** | macOS: productivity; Windows: learning + experimentation |

---

## **Bottom Line 🎯**

**Same Heart, Different Ceiling**

Both projects share the same elegant core idea: an AI tutor buddy that reads your screen and points at UI elements to guide you. But Windows took that foundation and built a full **learning companion platform** on top, while macOS kept it **simple and pristine** as a focused productivity tool.
