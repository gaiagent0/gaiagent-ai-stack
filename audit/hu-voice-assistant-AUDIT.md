# hu-voice-assistant — Backend Deep Audit

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\hu-voice-assistant`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Verzió:** v2.3.0 · **Státusz:** WIP / aktív fejlesztés
**Összefüggés:** gaiagent0 repócsalád auditja — magyar hangsegéd platform

---

## 1. Architektúra (Docker Compose microservice)

| Service | Port | Szerep |
|---|---|---|
| `redis` | 6379 | Session store (256MB, allkeys-lru) |
| `tts-server` | 7860 | Dual TTS: Piper (gyors) + F5-TTS (voice clone) |
| `builder` | 8001 | Assistant Builder API: STT + LLM + RAG + Admin export |
| `chat` | 8002 | Chat proxy + Redis session |
| `frontend` | 3010 | Next.js 15 (App Router, i18n hu/en) |
| `runtime` | 8000/8003 | Standalone exported assistant (numpy RAG + TTS) |

**Plusz modulok:** `hu-reader` (magyar felolvasó, F5 chunked), `npu-voice-cloner` (Piper QNN + F5 ONNX pipeline — NPU kísérlet).

---

## 2. STT (ASR)

| Tény | Érték |
|---|---|
| Modell | `sarpba/whisper-hu-large-v3-turbo-finetuned` (HuggingFace transformers) |
| Mód | `STT_MODE=local` (default) — **NEM Groq API** (a README említi Groq Whisper-t is, de a CLAUDE.md szerint lokális HF) |
| Nyelv | `WHISPER_LANGUAGE=hu` |
| Helyi NPU? | ❌ Nincs — CPU/GPU transformers. A `npu-voice-cloner` NPU kísérlet, de az STT nincs NPU-ra hozva |

**GenieX átjárhatóság:** A Whisper-Base QNN (GenieX-stack) elméletileg átvehetné az STT-t, de:
- A `whisper-hu-large-v3-turbo-finetuned` **HU-finomhangolt**, WER ~10-15% — a Whisper-Base QNN multilingual csak ~60-85% WER HU-n.
- **Nem éri meg váltani** — a HU-finomhangolt Whisper jobb. Ha NPU kell, inkább a `whisper-hu-large-v3-turbo` QNN exportja lenne a cél (nem a Base QNN).

---

## 3. TTS

| Motor | Modell | Sebesség (CPU) | Voice Cloning |
|---|---|---|---|
| **Piper** (default) | `hu_HU-anna-medium.onnx` | ~190-300ms | ❌ |
| **F5-TTS** | `sarpba/F5-TTS_V1_hun_v2` | ~85-260s/batch | ✅ (≤150 char) |

- `TTS_ENGINE=auto` → Piper ha elérhető, F5 fallback
- **Ez PONT UGYANAZ a Piper hu_HU-anna, mint a meetcore-ben!** (közös hangszín)
- F5-TTS CPU-n ARM64 (Snapdragon X Elite): 1 batch ~170s → csak rövid válaszra
- NPU: F5-TTS ONNX→QNN konverzió tervben (DakeQQ/F5-TTS-ONNX) — még nincs kész

**GenieX átjárhatóság:** Piper már bent van (nem kell váltani). F5 marad CPU-n, NPU konverzió nyitott feladat.

---

## 4. LLM

| Provider | Model | API Key |
|---|---|---|
| groq (default) | groq/llama-3.3-70b-versatile | GROQ_API_KEY |
| gemini | gemini/gemini-2.0-flash-exp | GEMINI_API_KEY |
| openrouter | openrouter/anthropic/claude-3-haiku | OPENROUTER_API_KEY |
| anthropic | anthropic/claude-sonnet-4-6 | ANTHROPIC_API_KEY |
| ollama | ollama/qwen2.5:14b | (lokális) |
| nexa | openai/nexa-active-model | (lokális) |

- **LiteLLM multi-provider routing** (`llm_service.py`)
- Fallback chain: `groq,gemini,ollama,openrouter,anthropic`
- **NINCS GenieX** provider (csak `nexa` legacy)
- Tool-calling: LiteLLM támogatja, de a builder chat valószínűleg sima text completion

**GenieX átjárhatóság:** KÖNNYŰ. A `nexa` provider helyett `geniex` (Qwen3-4B, :18181) — egy LiteLLM config sor. A Qwen3-4B (GenieX) bekerülhet a fallback chain-be is.

---

## 5. RAG

- ChromaDB + `BAAI/bge-m3` embedding (multilingual)
- Builder: BM25 + dense RRF EnsembleRetriever
- Runtime: numpy cosine similarity (exportált embeddings)
- Ingest: PDF/DOCX/TXT/CSV/URL

**GenieX átjárhatóság:** Nem érintett — RAG független a futtatótól.

---

## 6. Tech Debt

1. **STT nincs NPU-n** — CPU/GPU transformers (lassú, vagy Groq felhő)
2. **F5-TTS CPU-n lassú** (~170s batch) — NPU konverzió tervben, nincs kész
3. **LLM: nincs GenieX** — csak felhős (groq/gemini/anthropic) + ollama + nexa legacy
4. **`nexa` provider** — legacy NexaAI, GenieX-re váltandó (ha helyi NPU LLM kell)
5. **GenieAPIService :8912** — a CLAUDE.md "Jövőbeli irányok" említi (llama3.1-8b), de ez OFFLINE/legacy
6. **Dependency ütközések** Python 3.13-on (pydantic, chromadb, langchain, litellm) — izolált venv szükséges
7. **Test coverage**: rag_service ~37%, knowledge.py ~28% (BackgroundTask miatt nehezen tesztelhető)

---

## 7. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| API key kezelés | `.env` (Docker) — GROQ/GEMINI/ANTHROPIC kulcsok | ⚠️ .env-ben, de .gitignore védi |
| CORS | `*` (fejlesztés) | ⚠️ Production előtt szűkítendő |
| Hardcoded secret | Nincs található | ✅ Jó |
| Redis session | TTL 24h, allkeys-lru | ✅ Jó |

---

## 8. GenieX átjárhatóság — ÖSSZEGZÉS

| Réteg | Átjárható GenieX-re? | Munka | Megéri? |
|---|---|---|---|
| **LLM** | ✅ Igen (`nexa` → `geniex` Qwen3-4B) | 1 sor LiteLLM config | ✅ Igen (helyi NPU LLM) |
| **STT** | ⚠️ Elméletileg (Whisper-Base QNN) | Közepes | ❌ Nem — a HU-finomhangolt Whisper jobb WER |
| **TTS** | ✅ Piper már bent | 0 (Piper ready) | ✅ Már GenieX-kompatibilis |
| **RAG** | ✅ Nem érintett | 0 | ✅ Nem kell |

**Javaslat:** Az LLM provider-t érdemes GenieX Qwen3-4B-re váltani (helyi NPU inferencia, nincs felhő költség). Az STT maradjon HU-finomhangolt Whisper (vagy ha NPU kell: whisper-hu-large-v3-turbo QNN export). A TTS (Piper hu_HU-anna) már GenieX-kompatibilis.

---

## 9. Kapcsolódás a meetcore-hoz

- **KÖZÖS:** Piper `hu_HU-anna-medium` TTS (azonos hangszín!), F5-TTS voice clone, magyar i18n
- **KÜLÖNBSÉG:** a meetcore meeting-assistant (ASR→összefoglaló), a hu-voice-assistant általános hangsegéd platform (builder + RAG + standalone runtime)
- **Átfedés:** mindkettő használhatja a GenieX Qwen3-4B-t LLM-nek, és a Piper hu_HU-anna-t TTS-nek → **közös skill: `hungarian-tts` (Piper) + `geniex-npu` (Qwen3-4B)**

---

## 10. Összegzés

**Állapot:** Érett v2.3.0 platform, aktív fejlesztés alatt. Szép architektúra (microservice + standalone runtime).
**GenieX potenciál:** Közepes — LLM könnyen váltható (nexa→geniex), STT maradjon HU-Whisper, TTS már kész.
**Tech debt:** NPU hiány (STT/F5 CPU-n), legacy nexa provider, rate-limit veszély felhős LLM-knél.
**Döntés:** Érdemes a `gaiagent-voice-audit` skillbe beemelni — a Piper hu_HU-anna + F5-TTS + GenieX LLM kombináció közös pont a meetcore-ral.

**Fájlok olvasva:** README.md, CLAUDE.md (v2.3.0), assistant-runtime/app/runtime.py (structure)
**Nem olvastam mélyebben:** assistant-builder/app/services/*.py (a CLAUDE.md fedte a logicát)
