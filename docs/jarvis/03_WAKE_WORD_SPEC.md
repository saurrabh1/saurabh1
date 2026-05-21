# Jarvis — Wake-Word Trigger
## Hardware / Software Technical Specification

---

## The Core Problem

Standard voice assistants (Siri, Google Assistant) own the OS-level microphone daemon. A personal app does not get that privilege. The challenge is:

- Keep a microphone listener alive when **screen is off and app is backgrounded**
- Use **near-zero CPU/battery** while listening
- Detect a **custom wake word** ("Hey Jarvis") with low false-positive rate
- Seamlessly hand off to the full STT pipeline upon detection

---

## Recommended Solution: Picovoice Porcupine

**Why Porcupine over alternatives:**

| | Porcupine | OpenWakeWord | Snowboy | Google HOTW |
|---|---|---|---|---|
| On-device (no cloud) | ✅ | ✅ | ✅ | ❌ |
| Custom wake word | ✅ | ✅ (limited) | ✅ | ❌ |
| Mobile SDK | ✅ (Android + iOS) | ❌ (Python only) | Deprecated | ✅ |
| Screen-off support | ✅ | ❌ | ❌ | OS-only |
| CPU usage | < 1% | ~3-5% | ~2% | OS-level |
| False positive rate | Very low | Moderate | Moderate | Low |
| Free tier | 3 wake words | Fully free | Deprecated | N/A |

Porcupine runs a quantized neural net directly on the device's DSP/NPU. The "Hey Jarvis" custom model is trained via the Picovoice Console (web UI, ~5 minutes, free tier).

---

## Android Implementation

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   Android Device                          │
│                                                          │
│  ┌────────────────────────────────┐                      │
│  │  Foreground Service            │  ← runs 24/7         │
│  │  (JarvisWakeService.kt)        │                      │
│  │                                │                      │
│  │  AudioRecord (16kHz, mono PCM) │                      │
│  │       │                        │                      │
│  │       ▼                        │                      │
│  │  Porcupine Engine              │                      │
│  │  (hey_jarvis.ppn model)        │                      │
│  │       │                        │                      │
│  │       │ onWakeWordDetected()   │                      │
│  │       ▼                        │                      │
│  │  [Switch to full pipeline]     │                      │
│  └────────────────────────────────┘                      │
│             │                                            │
│             ▼                                            │
│  ┌──────────────────────┐                                │
│  │  Deepgram WebSocket  │                                │
│  │  STT Streaming       │──────► FastAPI Backend         │
│  └──────────────────────┘                                │
│                                                          │
│  [Screen OFF, app backgrounded — this still works]       │
└──────────────────────────────────────────────────────────┘
```

### Key Android Requirements

**1. Foreground Service with Persistent Notification**

Android kills background processes. A Foreground Service with a persistent notification (required by Android 8+) keeps the wake word listener alive.

```kotlin
// JarvisWakeService.kt
class JarvisWakeService : Service() {

    private lateinit var porcupine: Porcupine
    private lateinit var audioRecord: AudioRecord
    private var isListening = false

    override fun onCreate() {
        super.onCreate()
        startForeground(NOTIF_ID, buildNotification())  // required
        initPorcupine()
    }

    private fun initPorcupine() {
        porcupine = Porcupine.Builder()
            .setAccessKey(BuildConfig.PICOVOICE_KEY)
            .setKeywordPath("hey_jarvis.ppn")   // custom model asset
            .setSensitivity(0.7f)                // 0.0–1.0; higher = more sensitive
            .build(applicationContext)
    }

    private fun startListeningLoop() {
        val bufferSize = AudioRecord.getMinBufferSize(
            porcupine.sampleRate,
            AudioFormat.CHANNEL_IN_MONO,
            AudioFormat.ENCODING_PCM_16BIT
        )

        audioRecord = AudioRecord(
            MediaRecorder.AudioSource.VOICE_RECOGNITION,
            porcupine.sampleRate,       // 16000 Hz
            AudioFormat.CHANNEL_IN_MONO,
            AudioFormat.ENCODING_PCM_16BIT,
            bufferSize
        )

        audioRecord.startRecording()
        isListening = true

        // Runs on a low-priority background thread
        thread(priority = Thread.MIN_PRIORITY) {
            val pcmBuffer = ShortArray(porcupine.frameLength)
            while (isListening) {
                audioRecord.read(pcmBuffer, 0, pcmBuffer.size)
                val keywordIndex = porcupine.process(pcmBuffer)
                if (keywordIndex >= 0) {
                    onWakeWordDetected()
                }
            }
        }
    }

    private fun onWakeWordDetected() {
        // 1. Haptic feedback (subtle)
        vibrate(50)
        // 2. Play earcon (short "ding") via audio focus
        playWakeEarcon()
        // 3. Switch to full Deepgram streaming
        JarvisSTTPipeline.start(this)
    }
}
```

**2. Required Android Manifest Permissions**

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

<!-- Service declaration -->
<service
    android:name=".JarvisWakeService"
    android:foregroundServiceType="microphone"
    android:exported="false" />

<!-- Auto-restart on device boot -->
<receiver android:name=".BootReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
    </intent-filter>
</receiver>
```

**3. Battery Optimization — Critical Step**

Android's Doze mode will kill the service unless the user whitelists the app:

```kotlin
// Request battery optimization exemption (shown once to user)
fun requestBatteryOptimizationExemption(context: Context) {
    val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
    if (!pm.isIgnoringBatteryOptimizations(context.packageName)) {
        val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
            data = Uri.parse("package:${context.packageName}")
        }
        context.startActivity(intent)
    }
}
```

---

## Full Audio Pipeline (Post-Wake-Word)

Once wake word is detected, the system transitions to the full streaming pipeline:

```
Phase 1: Wake Detection (always on, ~0.8% CPU)
─────────────────────────────────────────────
Porcupine (on-device) → detects "Hey Jarvis"
    ↓ (detection event, ~200ms latency)

Phase 2: Activation (< 300ms)
─────────────────────────────────────────────
Play wake earcon → Start Deepgram WebSocket
    ↓

Phase 3: STT Streaming (utterance capture)
─────────────────────────────────────────────
Deepgram streams PCM 16kHz → returns interim + final transcripts
End-of-speech detected (silence > 1.5s) → close STT stream
    ↓ final transcript text

Phase 4: Agent Processing (1-3s)
─────────────────────────────────────────────
FastAPI → LangGraph Agent Manager → sub-agent → response text
    ↓

Phase 5: TTS + Audio Output (< 1s)
─────────────────────────────────────────────
Response text → OpenAI TTS stream → audio played to earbuds
    ↓

Return to Phase 1 (Porcupine resumes listening)
```

**Target end-to-end latency breakdown:**

| Phase | Target |
|-------|--------|
| Wake word detection | 200ms |
| STT (streaming, real-time) | ~0ms delay (streaming) |
| Network to backend | 50-100ms |
| LLM intent classification | 300-500ms |
| Sub-agent execution | 500ms-2s (web search longer) |
| TTS generation | 300-500ms |
| **Total (simple query)** | **~2-3 seconds** |
| **Total (web search)** | **~4-5 seconds** |

---

## Desktop / PC Support (Secondary Platform)

For the PC use case (screen locked, headphones connected):

```python
# desktop_wake.py — runs as system daemon
import pvporcupine
import pyaudio
import numpy as np

def run_wake_daemon():
    porcupine = pvporcupine.create(
        access_key=PICOVOICE_KEY,
        keyword_paths=["hey_jarvis_mac.ppn"],   # platform-specific model
        sensitivities=[0.7]
    )

    pa = pyaudio.PyAudio()
    stream = pa.open(
        rate=porcupine.sample_rate,
        channels=1,
        format=pyaudio.paInt16,
        input=True,
        frames_per_buffer=porcupine.frame_length
    )

    print("[Jarvis] Listening for wake word...")
    try:
        while True:
            pcm = stream.read(porcupine.frame_length, exception_on_overflow=False)
            pcm_array = np.frombuffer(pcm, dtype=np.int16)
            result = porcupine.process(pcm_array)
            if result >= 0:
                on_wake_word_detected()
    finally:
        porcupine.delete()
        stream.close()
        pa.terminate()
```

**Install as a system service (macOS):**
```bash
# launchd plist at ~/Library/LaunchAgents/com.jarvis.wake.plist
# Starts on login, restarts on crash
```

**Install as a Windows service:**
```bash
# NSSM (Non-Sucking Service Manager)
nssm install JarvisWake python C:\jarvis\desktop_wake.py
nssm set JarvisWake Start SERVICE_AUTO_START
```

---

## Custom Wake Word Training

1. Go to [Picovoice Console](https://console.picovoice.ai/)
2. Create account (free tier: 3 wake words)
3. New Wake Word → type "Hey Jarvis"
4. Select language: English
5. Download platform-specific `.ppn` file:
   - `hey_jarvis_android.ppn`
   - `hey_jarvis_ios.ppn`
   - `hey_jarvis_mac.ppn`
   - `hey_jarvis_windows.ppn`
6. Bundle in app assets
7. Set sensitivity `0.65-0.75` (balance: responsiveness vs false positives)

---

## Earbud Audio Routing

The earbuds do **not** run the wake-word engine — they are output-only for TTS and audio feedback. The phone handles all processing.

```
Phone microphone     → wake word detection + STT input
Phone speaker/BT     → TTS audio output → earbuds via A2DP Bluetooth

For earbuds with onboard mic (e.g. Galaxy Buds, AirPods):
  → The phone app can optionally route mic input through BT HFP profile
  → Deepgram can receive the earbud mic stream instead of phone mic
  → Better voice isolation in noisy environments
  → Requires: audioRecord.setPreferredDevice(bluetoothHeadsetDevice)
```

---

## Sensitivity Tuning Guide

| Sensitivity | Behavior | Best For |
|------------|----------|----------|
| 0.4 – 0.55 | Rarely triggers accidentally, may miss soft speech | Quiet environments |
| 0.65 – 0.75 | Good balance, recommended starting point | General use |
| 0.80 – 0.90 | Very responsive, higher false positive rate | Noisy environments |

Start at `0.7` and tune down if too many accidental triggers.
