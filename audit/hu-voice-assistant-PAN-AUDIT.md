# hu-voice-assistant — Pan Audit (2026-08-18)

## Architektúra

Docker Compose microservice stack (5 szolgáltatás):

| Service | Port | Funkció |
|---------|------|---------|
| `redis` | 6379 | Session store (256MB, allkeys-lru, no persistence) |
| `tts-server` | 7860 | Dual TTS: Piper (gyors) + F5-TTS (voice cloning) |
| `builder` | 8001 | Builder API: STT + LLM + RAG (BM25+RRF) + CRUD + Admin export |
| `chat` | 8002 | Chat API: proxy + Redis session |
| `frontend` | 3010 | Next.js UI |

Standalone runtime: `docker-compose.runtime.yml` — exportált asszisztens futtatás.

---

## ASR (Speech-to-Text)

| Komponens | Részletek |
|-----------|-----------|
| **Modell** | `sarpba/whisper-hu-large-v3-turbo-finetuned` — magyar fine-tune, large-v3-turbo |
| **Lib** | `faster-whisper==1.1.0` (CTranslate2, 4× gyorsabb mint PyTorch) |
| **Módok** | `groq` (cloud, default) · `local` (auto-detect) · `local-faster` · `local-hf` |
| **NPU** | ❌ NINCS — CPU/GPU only, nincs onnxruntime-qnn, nincs NexaAI |

**GenieX kompatibilitás: ❌ NEM kompatibilis.** A Whisper large-v3-turbo egy NAGY modell (~1.5GB), nem fér el a Whisper-Base QNN keretben. Az ASR cseréje jelentős munkát igényelne (modell csere + finetune).

---

## TTS (Text-to-Speech)

| Engine | Latency (CPU) | Voice Cloning | Modell |
|--------|---------------|---------------|--------|
| **Piper** | ~300ms | ❌ | `hu_HU-anna-medium.onnx` |
| **F5-TTS** | ~85-260s/batch | ✅ | `sarpba/F5-TTS_V1_hun_v2` |

- **Választás**: `TTS_ENGINE=auto` (Piper ha elérhető, F5 fallback)
- **Voice cloning**: F5-TTS + ref audio upload (WebM/MP3/OGG/WAV → WAV 24kHz)
- **Chunking**: `F5_CHUNK_CHARS` env, `_split_sentences()` + `_concat_wavs()`

**GenieX kompatibilitás: ⚠️ RÉSZLEGES.** Piper ONNX → QNN konverzió lehetséges (DakeQQ/F5-TTS-ONNX). F5-TTS-nek ONNX export kellene + QNN konverzió.

---

## LLM Provider-ek

Fájl: `assistant-builder/app/services/llm_service.py` (LiteLLM alapú)

| Provider | Default Model | API Key |
|----------|--------------|---------|
| **groq** (default) | `groq/llama-3.3-70b-versatile` | `GROQ_API_KEY` |
| gemini | `gemini/gemini-2.0-flash-exp` | `GEMINI_API_KEY` |
| openrouter | `openrouter/anthropic/claude-3-haiku` | `OPENROUTER_API_KEY` |
| anthropic | `anthropic/claude-sonnet-4-6` | `ANTHROPIC_API_KEY` |
| ollama | `ollama/qwen2.5:14b` | (lokális) |
| **nexa** | `openai/nexa-active-model` | (lokális, `NEXA_API_BASE`) |

- **Fallback chain**: `groq → gemini → ollama → openrouter → anthropic`
- **RAG**: ChromaDB + BM25 + dense (BAAI/bge-m3) EnsembleRetriever

**GenieX kompatibilitás: ✅ KÖNNYŰ.** A `nexa` provider már létezik, `NEXA_API_BASE=http://host.docker.internal:18181/v1` — csak át kell irányítani a GenieX URL-jére (`http://host.docker.internal:18181/v1`). A GenieX Qwen3-4B kisebb mint a cloud modellek, de működik.

---

## GenieX Migrációs Összefoglalás

| Komponens | Átjárható? | Munkaigény |
|-----------|-----------|------------|
| **LLM** | ✅ Igen | Könnyű — Nexa provider átirányítás |
| **ASR** | ❌ Nem | Nehéz — Whisper large-v3-turbo → Base QNN, finetune kellene |
| **TTS** | ⚠️ Részleges | Közepes — Piper QNN konverzió lehetséges, F5 ONNX export kell |

---

## Tech Debt & Biztonság

| Téma | Állapot | Kockázat |
|------|---------|----------|
| **CORS `*`** | builder + chat | 🔴 Production előtt szűkíteni |
| **API key** | env vars (Docker) | 🟢 Standard gyakorlat |
| **Redis** | no persistence | 🟢 Szándékos (session cache) |
| **GenieAPIService** | CLAUDE.md #336 | 🟡 Legacy említés, frissítendő |
| **F5-TTS CPU** | ~170s/batch ARM64 | 🟡 Lassú, NPU konverzió kellene |

---

## Megjegyzések

- A `npu-voice-cloner/` mappa létezik — Piper QNN pipeline (`piper_anna_qnn.bin` kész), de ez egy külön projektrész, nem integrált a fő stackbe.
- A `hu-reader/` egy külön magyar felolvasó app (port 8003), F5-TTS chunked.
- A `_archive/` mappa elavult fájlokat tartalmaz.
- Solar4 Bot D a mély backend auditot csinálja párhuzamosan — ez az áttekintő nézet.
