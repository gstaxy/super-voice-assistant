# Security & Privacy Audit Report

**Repository:** `super-voice-assistant`
**Auditor:** Claude Code (automated analysis)

---

## Overall Verdict: Safe to Run

No malicious code, data exfiltration, or adversarial scripts were found. The app is a legitimate macOS voice assistant with proper security practices.

---

## What Stays Local (No Network)

- **WhisperKit & Parakeet transcription** — fully on-device, no data leaves your machine
- **Transcription history** — stored locally at `~/Documents/SuperVoiceAssistant/transcription_history.json` (last 100 entries, plain JSON)
- **Statistics** — local file at `~/Documents/SuperVoiceAssistant/transcription_stats.json`
- **Audio recordings** — kept in memory only, never written to disk, cleared after use
- **Screen recordings** — temporarily saved to Desktop, deleted after transcription
- **Preferences** — stored in UserDefaults (engine selection, audio devices)
- **ML models** — cached in `~/Documents/huggingface/` and `~/Documents/FluidAudio/`

---

## What Goes to the Network

All external communication is to **Google Gemini API only** — no other servers:

| Feature | Endpoint | Data Sent |
|---------|----------|-----------|
| Gemini audio transcription (Cmd+Opt+X) | `generativelanguage.googleapis.com` REST | Base64 WAV audio |
| Video transcription (Cmd+Opt+C) | Same REST endpoint | Base64 video file |
| Text-to-speech (Cmd+Opt+S) | Same host, WebSocket | Selected text |
| Model downloads (one-time) | Hugging Face | Nothing uploaded |

These features are **opt-in** — they only activate when you press the specific shortcut. The WhisperKit/Parakeet paths (Cmd+Opt+Z) are fully offline.

---

## No Malicious Patterns Found

- **No analytics, telemetry, or tracking** of any kind
- **No git hooks** active (all are `.sample` defaults)
- **No build scripts** that execute hidden code
- **No launch agents/daemons** or persistence mechanisms
- **No obfuscated code** or encoded URLs
- **No iCloud/CloudKit** sync
- **No Keychain access**
- **No shell command injection** vectors
- **No data sent to unknown servers**

---

## Dependencies (All Reputable)

| Dependency | Source | Purpose |
|------------|--------|---------|
| `KeyboardShortcuts` (v1.8.0) | sindresorhus (GitHub) | Keyboard shortcut handling |
| `WhisperKit` (v0.13.0) | argmaxinc (GitHub) | On-device speech recognition |
| `FluidAudio` (v0.7.9) | FluidInference (GitHub) | Parakeet transcription engine |

Transitive dependencies (pulled in automatically):
- `swift-argument-parser` (v1.6.1, Apple)
- `swift-collections` (v1.2.1, Apple)
- `swift-transformers` (v0.1.15, Hugging Face)
- `Jinja` (v1.2.4, johnmai-dev)

---

## Detailed File-Level Findings

### Network Communication

**Gemini REST API** (`SharedSources/GeminiAudioTranscriber.swift`, `SharedSources/VideoTranscriber.swift`):
- Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- Authentication: API key passed in URL query parameter
- Data transmitted: base64-encoded audio/video + prompt text
- Response: transcription text only

**Gemini WebSocket** (`SharedSources/GeminiAudioCollector.swift`):
- Endpoint: `wss://generativelanguage.googleapis.com/ws/...BidiGenerateContent`
- Data transmitted: user-selected text for TTS
- Response: audio chunks for playback

**WhisperKit Model Downloads** (`Sources/WhisperModelDownloader.swift`):
- Source: Hugging Face (`argmaxinc/whisperkit-coreml`)
- Download only — no user data uploaded

**Parakeet Model Downloads** (`SharedSources/ParakeetTranscriber.swift`):
- Source: Hugging Face (`FluidInference/parakeet-tdt-*`)
- Download only — no user data uploaded

### System Resource Access

| Resource | Files | Purpose | Assessment |
|----------|-------|---------|------------|
| Microphone | `AudioTranscriptionManager.swift`, `GeminiAudioRecordingManager.swift` | Audio input for speech recognition | Legitimate, uses proper `AVCaptureDevice.requestAccess()` |
| Clipboard | `main.swift` | Copy/paste transcribed text | Legitimate, preserves and restores original clipboard |
| Keyboard Events | `main.swift` | Global hotkey detection | Legitimate, uses `NSEvent.addGlobalMonitorForEvents` |
| File System | Multiple files | Store history, models, temp recordings | Legitimate, app-specific directories only |

### Shell/Process Execution

| File | Command | Purpose |
|------|---------|---------|
| `ScreenRecorder.swift` | `/usr/bin/env ffmpeg` | Screen recording with audio capture |
| `scripts/generate_xcode_icon_assets.sh` | `sips`, `iconutil` | Icon asset generation (development only) |
| `scripts/generate_macos_icns_bundle.sh` | `sips`, `iconutil` | macOS icon bundle (development only) |

No arbitrary command execution, no user-input-driven shell commands.

### Data Persistence

| Data | Location | Format | Lifecycle |
|------|----------|--------|-----------|
| Transcription history | `~/Documents/SuperVoiceAssistant/transcription_history.json` | Plain JSON (max 100 entries) | Persists across sessions |
| Transcription stats | `~/Documents/SuperVoiceAssistant/transcription_stats.json` | Plain JSON | Persists across sessions |
| WhisperKit models | `~/Documents/huggingface/models/argmaxinc/whisperkit-coreml/` | Binary CoreML | Persists (cached) |
| Parakeet models | `~/Documents/FluidAudio/models/` | Binary CoreML | Persists (cached) |
| Model metadata | Model folder `.download_metadata.json` | JSON | Persists (cached) |
| Preferences | System UserDefaults | Binary plist | Persists across sessions |
| Audio recordings | In-memory only | Float array | Cleared after each use |
| Screen recordings | `~/Desktop/screen-recording-*.mp4` | MP4 | Deleted after transcription |

### API Key Management

- Loaded from `.env` file or `GEMINI_API_KEY` environment variable
- `.env` is properly listed in `.gitignore`
- Not hardcoded in source code
- Loaded into process memory only (not persisted elsewhere)

---

## Minor Observations (Not Security Issues)

1. **Console logging** — Transcription text is printed via `print()` statements, visible in Terminal during the session
2. **Plain JSON history** — Transcription history is not encrypted, protected by standard macOS file permissions
3. **Notifications** — Transcription text shown in macOS notifications (truncated to 100 characters)

---

## Recommendations for Maximum Privacy

If you want a **fully offline, zero-network experience**:

1. Use only the **WhisperKit or Parakeet engine** (Cmd+Opt+Z) — these run entirely on-device
2. Avoid the Gemini shortcuts (Cmd+Opt+X, Cmd+Opt+S, Cmd+Opt+C) — these send data to Google
3. Do not set a `GEMINI_API_KEY` — the Gemini features will simply fail gracefully
4. Transcription history at `~/Documents/SuperVoiceAssistant/` can be periodically deleted if desired

---

## Conclusion

This is a clean, legitimate macOS voice assistant application. All network communication is to documented Google APIs, triggered only by explicit user action. The offline transcription paths (WhisperKit, Parakeet) involve zero network activity. There are no hidden data collection mechanisms, no background processes, no persistence beyond expected app data, and no adversarial code.
