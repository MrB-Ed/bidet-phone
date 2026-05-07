# Bidet Phone — local-Gemma Bidet AI for Pixel 8 Pro

This repo is the work-track for taking Bidet AI fully on-device on Mark Barnett's
Pixel 8 Pro using **Gemma 4 E4B + Google AI Edge SDK + LiteRT-LM**, with the
existing Bidet cleanup pipeline (RAW / CLEAN / ANALYSIS / FORAI tabs) bolted on.

**Status:** plan stage. No code yet. See [BIDET_PLAN.md](./BIDET_PLAN.md) for the
research + decisions Claude already locked in.

---

## TL;DR for an AI builder reading this

You are being handed this README with a prompt like *"build me an Android app that
does X."* Read this whole README + BIDET_PLAN.md, then proceed.

### What to build
A **fork** of Google's open-source [`google-ai-edge/gallery`](https://github.com/google-ai-edge/gallery)
app, morphed into a focused capture tool with these tabs (instead of Gallery's
generic chat):

1. **RAW** — Audio Scribe (real-time on-device transcription) + raw text capture.
2. **CLEAN** — Same transcript, cleaned: typos fixed, hesitations removed,
   sentences merged. Done by Gemma 4 E4B locally.
3. **ANALYSIS** — Optional: Gemma 4 generates a short bullet summary + flags any
   action items.
4. **FORAI** — Same content, prepared for ingest by another AI: stripped of
   tangents, structured headings, key-value pairs where useful.

### Constraints (non-negotiable)
- **All inference local.** No cloud Gemini API calls. Use `AICore` + Gemma 4 E4B
  via the AI Edge SDK / LiteRT-LM. Pixel 8 Pro NPU/GPU only.
- **No analytics. No telemetry.** Bidet captures Mark's classroom voice — never
  ships off device unless explicitly sent to the configured webhook.
- **Apex-master architecture.** Phone is a capture client, not the canonical
  store. After cleanup, payload POSTs to `https://ag.thebarnetts.info/ingest/bidet`
  with header `X-AG-KEY: <secret>`. (Endpoint to be added to TP3 stack — not
  blocking the build, the app can hold local until endpoint exists.)
- **MIT license.** Sideload via Obtainium — don't publish to Play Store.

### Reference apps to read first
- `google-ai-edge/gallery` — fork target. Has Audio Scribe + Gemma 4 E4B already.
- `RafhaanShah/TaskerHealthConnect` — clean Kotlin patterns for Android sensors.
- Bidet AI desktop (live at https://bidetai.thebarnetts.info) — UX reference for
  the four output tabs.

### Wire format for `/ingest/bidet` (planned)
```json
{
  "timestamp": "<ISO8601 UTC>",
  "device": "Pixel 8 Pro",
  "source": "bidet_phone",
  "session_id": "<uuid>",
  "raw": "<full transcript>",
  "clean": "<cleaned transcript>",
  "analysis": "<bullet summary + action items>",
  "forai": "<structured for downstream AI>"
}
```

### What's NOT in scope for v0.1
- Multi-user / login. Single-user app, Mark only.
- Cloud sync between phone instances.
- Background continuous capture. v0.1 = button-press to start session.
- iOS port.

---

## Pointers
- The full plan + decisions: [BIDET_PLAN.md](./BIDET_PLAN.md)
- TP3 stack architecture (the Apex-master end of the pipeline):
  https://github.com/MrB-Ed/tp3-neural-stack
- Cursor's parallel build for the Health Connect bridge (similar architecture):
  https://github.com/MrB-Ed/tp3-health-bridge
- Bidet AI desktop (current product): https://bidetai.thebarnetts.info

---

## License

MIT. See [LICENSE](./LICENSE).
