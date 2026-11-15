# Bidet Phone: historical Android planning notes

This repository preserves an early plan for an Android voice-capture and transcript-cleanup app. It is not a working app release or a current implementation brief.

**For implementation code, start with [Bidet AI](https://github.com/MrB-Ed/bidet-ai).** That public repository is a curated code extract, not a complete installable application.

Separately, [Bidet AI Voice for Android](https://github.com/MrB-Ed/bidet-ai-voice-android-release) documents a signed Android tester preview with cloud-provider options. That release lane is distinct from the curated on-device Gemma extract. Its release and device-test claims are project-reported; this documentation review did not test the APK or verify the current phone build.

## What the early plan explored

- Capturing speech and processing transcripts on an Android phone.
- Separating raw text, cleaned text, analysis, and text prepared for other tools.
- Comparing a reference-app fork with a purpose-built interface.
- An optional explicit export path to a separate personal system.

The original plan considered model, runtime, and hardware choices explored at that time. Its claims and estimates are historical research notes, not current compatibility, performance, or privacy guarantees. See [BIDET_PLAN.md](BIDET_PLAN.md) for the planning context.

## License and attribution

This repository contains an [MIT LICENSE](LICENSE) for its material. Referenced projects and models retain their own licenses and terms; the MIT file does not relicense upstream code or model weights. The implementation showcase above documents its separate license and attribution.
