# Troubleshooting: super-voice-assistant on macOS 26

## Startup crash (SIGTRAP in CoreMedia)

**Symptom:** App crashes immediately at `AVCaptureDevice.requestAccess` during init.

**Cause:** macOS 26 crashes when CoreMedia/AVFoundation APIs are called too early in the app lifecycle (during `applicationDidFinishLaunching`).

**Fix:** Defer `setupAudioEngine()` and `requestMicrophonePermission()` to the next run loop cycle using `DispatchQueue.main.async` in both `AudioTranscriptionManager.init()` and `GeminiAudioRecordingManager.init()`. This fix was applied to the codebase.

## Paste not working (text saved to history but not pasted)

**Symptom:** Transcription completes, logs show "Paste command sent" and correct target app, but nothing is pasted.

**Cause:** The app uses `CGEvent.post(tap: .cghidEventTap)` to simulate Cmd+V. This requires macOS Accessibility permission. Without it, the keystroke is silently ignored.

**What didn't work:**
- Adding the built binary (`.build/arm64-apple-macosx/debug/SuperVoiceAssistant`) to Accessibility — macOS invalidates permission on every rebuild (unsigned binary, new hash)
- Adding `/usr/bin/swift` to Accessibility — macOS doesn't associate `swift run` child processes with the swift binary's permissions
- Using AppleScript (`tell "System Events" to keystroke "v"`) — same permission issue, but with an explicit error instead of silent failure
- Calling `AXIsProcessTrustedWithOptions` with prompt — crashes on macOS 26 (AppKit metadata resolution bug)

**What worked:** Add your **terminal app** (e.g. Terminal.app, iTerm, Warp) to **System Settings > Privacy & Security > Accessibility**. That's it. No other entries needed.

## Gemini API key

Not required. The app runs fine without it. WhisperKit/Parakeet (Cmd+Opt+Z) work fully offline. Only the Gemini shortcuts (Cmd+Opt+X, Cmd+Opt+S, Cmd+Opt+C) need the key.
