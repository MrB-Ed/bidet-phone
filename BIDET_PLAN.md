# Bidet AI → Local on Pixel 8 Pro — research + plan (2026-05-07)

**Status:** Plan stage, not started. To be picked up next G16 session.
**Trigger:** Gemini ("Jim") drafted an architecture for taking Bidet AI fully on-device on the Pixel 8 Pro. Mark asked Claude to vet it. First Claude pass was lazy and hedged on facts; Mark pushed back; Claude researched and confirmed Jim was largely right.

## TL;DR
- Gemma 4 + Google AI Edge stack is real and viable on Pixel 8 Pro **today**.
- Google already ships a working reference app (**AI Edge Gallery**) on the Play Store that runs Gemma 4 E4B locally with on-device transcription. **Most of Bidet AI's capture+inference loop already exists in that app.**
- The cheapest experiment is "install the Gallery, test if Audio Scribe + Gemma 4 E4B can clean a real class transcript at the quality bar Bidet desktop already hits — and measure tokens/sec + thermal sustain."
- If yes → either fork the open-source Gallery app or build a thinner purpose-built shell on the AI Edge SDK. Both paths sized below.
- **Sync architecture decided: Apex-master with an offline phone buffer.** Phone captures + cleans locally, posts to Apex on Wi-Fi/cell when reachable, queues when not. TP3 stays canonical.
- **What this adds vs OMI:** real-time, in-class sidebar generation. OMI capture is async and sometimes hours late (per `feedback_omi_wifi_only_sync_2026-05-05.md` — school cellular dead zones queue conversations until evening Wi-Fi). Bidet-on-phone runs the clean-up pass immediately, so a student-facing outline is ready within seconds of the bell. Different tool, different latency band.

## What was verified (sources at bottom)

### Gemma 4 — confirmed real, launched 2026-04-02
- Four sizes: **E2B** (~2.3B effective params), **E4B** (~4.5B), 26B MoE, 31B Dense.
- E2B/E4B are the **edge** variants — built for on-device.
- **Native audio input** confirmed on E2B/E4B (not just text+vision like Gemma 3).
- **128K context window** on E2B/E4B edge variants (the larger 26B A4B / 31B Dense models get 256K — don't conflate them).
- All sizes Apache 2.0. Available on Hugging Face, Kaggle, Ollama (`ollama pull gemma4:e2b` / `e4b`).

### LiteRT-LM — confirmed real
- Google's open-source on-device LLM inference framework (formerly TFLite-LM).
- Production framework powering Gemini Nano in Chrome, Chromebook Plus, Pixel Watch.
- Cross-platform: Android, iOS, Web, Desktop, Raspberry Pi.
- **Pixel 8 GPU is a tested target** — there's a public LiteRT-LM issue (#1850) where Gemma 4 E2B on Pixel 8 GPU was failing on 0.10.0 and a 0.10.1 patch fixed it. Means Pixel 8 / 8 Pro is on the maintained matrix.

### Google AI Edge Gallery — confirmed shipping
- Free app on Google Play. Open-source on GitHub: `google-ai-edge/gallery`.
- Runs **Gemma 4 E4B** (flagship) and **E2B** (mid-range) fully on-device, no cloud.
- Built-in features that overlap with Bidet AI:
  - **Audio Scribe** — real-time transcription + translation (the core Bidet capture loop).
  - **AI Chat with Thinking Mode** — step-by-step reasoning, useful for the "clean tangents into a sidebar" prompt.
  - **Agent Skills** — tool-use scaffolding (Wikipedia grounding, maps).
- Min OS: Android 12 / iOS 17. Pixel 8 Pro qualifies.

### AICore (Pixel-specific accelerator path)
- Real. Android system service that exposes on-device foundation models to apps.
- Google announced **Gemma 4 Developer Preview via AICore** at the Gemma 4 launch — meaning apps can call Gemma 4 through AICore rather than bundling weights themselves.
- Power/thermal advantage: NPU path through AICore is more efficient than CPU-only inference for long captures (50-min lecture without thermal throttling).

### What Jim got wrong
- **"ChromaDB (Mobile)"** — not a real product. Right answers for on-device vector storage on Android:
  - **ObjectBox** — first-party Android, HNSW, ~3MB binary, pure on-device, no cloud.
  - **sqlite-vec** / sqlite-vector — SQLite extension, cross-platform, ~30MB.
- Either works. ObjectBox is the cleaner Android-native pick.

### What Claude got wrong (first pass)
- Hedged on Gemma 4 dates instead of searching. Lazy. Mark called it out. Logged here so it doesn't happen again — when a date or product is past the training cutoff, **search first, hedge never**.

## Architecture — Bidet AI native Android (the build, if Mark greenlights)

| Component | Tool | Notes |
|---|---|---|
| Inference runtime | **LiteRT-LM** via **Google AI Edge SDK** | Already Pixel-tested |
| Model | **Gemma 4 E4B** (instruction-tuned), via AICore Developer Preview | E2B as fallback for thermal/RAM pressure |
| Transcription | **Audio Scribe path** (Gallery app) OR Android SpeechRecognizer offline + Gemma 4 native audio | Test both; native audio in Gemma 4 E4B may eliminate the separate STT step |
| App shell | **Fork of `google-ai-edge/gallery`** OR **purpose-built shell on AI Edge SDK** | See "Fork vs purpose-built" below — sized both. Either way: keep existing Bidet output channels (**RAW / CLEAN / ANALYSIS / FORAI**, per `project_bidet_ai_vision.md` + `feedback_bidet_routes_in_main_not_app.md`); don't redesign the channel model on the phone. Capture-context (history class / brain dump / clean-students-for-absent / generic) is a separate dropdown that selects the system prompt; output channels stay constant. |
| Local memory | **ObjectBox** (vector) + SQLite (metadata) | Only if Bidet needs to remember prior class notes between sessions. Skip in v1. |
| TP3 bridge | **Apex-master with phone offline buffer** | Phone POSTs cleaned outputs to `https://ag.thebarnetts.info/ingest/bidet` with `X-AG-KEY`. If unreachable (school cellular gaps), queue locally in WorkManager + flush on Wi-Fi. Apex `tp3_memories_local` stays canonical — phone is a smart capture client, not a master copy. |

## Sync architecture — DECIDED (was framed as open earlier; not actually open)
**Apex-master with phone offline buffer.** Phone is a smart capture client, not a master copy.

Reasons it isn't actually a coin-flip:
- TP3 lives on Apex; the whole stack philosophy is server-side robust (per `feedback_no_overpromise_phone_setup_2026-05-02.md` — phone setup breaks, server-side first).
- Phone storage is finite — Apex just cleared 128GB recently to reclaim headroom; the phone has no equivalent slack.
- Phones get lost / replaced / wiped; TP3 must not depend on a single device's filesystem.
- Existing `/ingest` infra already accepts POST + X-AG-KEY from many sources (Tasker, SMS Forward, Health Connect bridge, Gmail/Yahoo pollers); Bidet-on-phone is just one more producer.

**Mechanics:**
- Phone records → on-device STT → Gemma 4 E4B clean pass → POST to `https://ag.thebarnetts.info/ingest/bidet` with payload `{raw, clean, analysis, forai, capture_context, ts, device}` and header `X-AG-KEY`.
- If POST fails (cellular dead zone at St. Francis, OMI Wi-Fi-only sync pattern), enqueue in **Android WorkManager** with `NetworkType.CONNECTED` constraint + exponential backoff. Flush on next Wi-Fi.
- Local copy retained on phone for ~7 days as belt-and-suspenders, then auto-pruned. Phone is never the system-of-record beyond that window.

PD #1 alignment: TP3 row-count delta keeps flowing because every cleaned output ships to Apex. If the phone is offline, the queue protects the record. If the phone is destroyed, at most 7 days of unsync'd buffer is at risk — all earlier captures are already in `tp3_memories_local`.

## Cheapest experiment (Mark drives, 5 min setup + one class period)
1. Install **Google AI Edge Gallery** from the Play Store on the Pixel 8 Pro (Mark plans to drive this from AI Studio).
2. Download **Gemma 4 E4B** inside the app (one-time, several GB).
3. Open Audio Scribe → record a real history-class block (10-15 min is enough for quality, but run one full 50-min lecture for the thermal/sustain test).
4. Pipe the transcript into AI Chat with a Bidet-style prompt: *"Identify my tangents and move them to a Sidebars section. Output a clean, bulleted outline for a student who missed the class."*
5. **Measure four things, not just quality:**
   - **Quality** — diff Gemma 4 E4B output against a Gemini-API run of the same transcript through current Bidet. Mark eyeballs both.
   - **Tokens/sec sustained** over the full 50-min lecture (Gallery exposes this in dev info).
   - **Thermal throttle** — does the phone hot-throttle at any point? Note ambient + chassis temp at start and end.
   - **Cost crossover** — current cloud Gemini cost per Bidet call vs. ~$0 marginal local cost; multiply by Mark's daily call volume to confirm payback math (independent of quality).
6. Decide: does Gemma 4 E4B's local output clear the bar **and** sustain through a real classroom session?

If yes → pick fork vs purpose-built (next section), kick off build.
If no → stay on the Gemini API path, revisit when Gemma 4.x or hardware improves.

## Fork vs purpose-built — size both, don't default
**Fork `google-ai-edge/gallery`:**
- ✅ Capture loop + AI Edge SDK plumbing + Audio Scribe already work.
- ✅ Faster to first working APK (days, not weeks).
- ❌ Permanent upstream-tracking cost. Google iterates the Gallery aggressively; every release means a rebase or a divergence decision. Over a year that's non-trivial maintenance.
- ❌ Carries UI/feature surface Bidet doesn't need (Image, Maps, Wikipedia agent skills) — either rip out (more rebase pain) or hide.

**Purpose-built shell on AI Edge SDK:**
- ✅ Smaller surface, no upstream debt, UI matches existing Bidet exactly.
- ✅ Easier to keep RAW/CLEAN/ANALYSIS/FORAI channel parity with desktop.
- ❌ Have to re-implement the recording UI, model-download UX, session management. The Gallery's shell isn't trivial.
- ❌ Slower to first working APK.

**Decision criterion:** if Mark's expected use is "Bidet, only Bidet" forever, purpose-built is cheaper over 12+ months. If Mark wants to keep experimenting with other on-device models that the Gallery already supports (Llama, Phi-4, Qwen), fork wins. **Decide after the experiment, with the agent that builds it weighing in.**

## Build path (if experiment passes)

### Coding agent — A/B Cursor vs Jules, don't default
Earlier draft of this doc defaulted to Jules. Premature. Tonight (2026-05-06 evening) **Cursor cloud-agent shipped a complete Kotlin/Compose Android APK + WorkManager + tests + GH Actions in one shot** for the HC Bridge (`MrB-Ed/tp3-health-bridge` PR #1). That changes the calculus.

- **Cursor cloud-agent** — proven on a real Android+WorkManager build this week. Cloud-VM-PR loop matches the cursor-agent CLI pattern in `feedback_use_cursor_agent_cli_directly_2026-05-06.md`.
- **Jules** — Google-native, tightest Gemma/AICore docs proximity. Untested by us on a build this size.
- **Android Studio + Gemini in Android Studio** — local fallback if both async agents stall.

**Plan:** hand the same Bidet brief to Cursor and Jules in parallel, take the better APK. The HC Bridge precedent means Cursor is no longer just a generalist — it's already shipped Android.

### Brief skeleton (for whichever agent)
1. Clone `google-ai-edge/gallery` (if forking) or `cursor new-android-project` (if purpose-built).
2. Brief: "Build Bidet AI for Android. Output channels are **RAW / CLEAN / ANALYSIS / FORAI** — match the desktop app, do not redesign. Capture-context dropdown selects the system prompt (history class / brain dump / clean-for-absent-student / generic). Use Gemma 4 E4B via AI Edge SDK + AICore. On [Send to TP3], POST `{raw, clean, analysis, forai, capture_context, ts, device}` to `https://ag.thebarnetts.info/ingest/bidet` with `X-AG-KEY`. If POST fails, queue in WorkManager with `NetworkType.CONNECTED` + exponential backoff. Sideload-only, debug build, no Play Store."
3. Build → sideload to Pixel 8 Pro.
4. Once stable, decide on publish (probably keep personal).

### What Claude does
- Write the brief.
- Verify the resulting code by reading it (architect, not coder, per `feedback_claude_code_is_architect.md`).
- Wire the Apex-side `/ingest/bidet` endpoint if it doesn't exist yet.
- Never ask Mark to copy-paste anything (per `feedback_no_control_no_copypaste.md` and `feedback_use_cursor_agent_cli_directly_2026-05-06.md`).

## Files to cross-reference next session
- `project_bidet_ai_vision.md` — full Bidet product vision (desktop + Chrome ext + Android already on the roadmap, this is the Android leg).
- `project_honest_answers.md` — current Bidet state (Whisper + Gemini cloud).
- `project_local_llm_goals.md` — local-LLM independence goal; this advances it.
- `feedback_claude_code_is_architect.md` — Claude writes briefs, doesn't write Android code.
- `feedback_legacy_soil_closed_topics.md` — pattern for what NOT to relitigate (apply to Bidet decisions once made).

## Sources (verified 2026-05-07)
- Gemma 4 launch: https://blog.google/innovation-and-ai/technology/developers-tools/gemma-4/
- Gemma 4 model card: https://deepmind.google/models/gemma/gemma-4/
- LiteRT-LM overview: https://ai.google.dev/edge/litert-lm/overview
- AI Edge Gallery on Play Store: https://play.google.com/store/apps/details?id=com.google.ai.edge.gallery
- AI Edge Gallery source: https://github.com/google-ai-edge/gallery
- Pixel 8 GPU + Gemma 4 LiteRT-LM issue: https://github.com/google-ai-edge/LiteRT-LM/issues/1850
- ObjectBox on-device vector DB: https://objectbox.io/the-on-device-vector-database-for-android-and-java/
- sqlite-vec (asg017): https://github.com/asg017/sqlite-vec
- Gemma 4 model card (Google AI for Developers): https://ai.google.dev/gemma/docs/core/model_card_4
- Gemma 4 E4B on Hugging Face: https://huggingface.co/google/gemma-4-E4B
- Agentic Gemma 4 launch post: https://developers.googleblog.com/bring-state-of-the-art-agentic-skills-to-the-edge-with-gemma-4/

## Lesson logged
When asked to evaluate a technical claim that's past the training cutoff: **search first, hedge never.** Mark caught this and was right to. Don't fall back on "I can't verify" when WebSearch is one tool call away.
