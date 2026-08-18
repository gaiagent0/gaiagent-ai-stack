# snapdragon_AI — Audit

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\snapdragon_AI`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Verzió:** v1.0.0 (api_server.py verzió) · **Státusz:** Fejlesztés alatt?
**Összefüggés:** gaiagent0 repócsalád auditja — Python agent framework

---

## 1. Mi ez?

**Production-ready AI assistant API** (FastAPI) — Qualcomm NPU gyorsítású chat backend.
NEM csak docs, hanem **futtatható Python alkalmazás**: `python main.py --port 8000`.
Összetevők: API server + inference engine + model loader + conversation manager + analytics tracker + RAG (document_loader).

---

## 2. Architektúra

```
main.py → uvicorn → api_server:app (:8000, 127.0.0.1)
  ├─ ModelLoader (betölti a QNN modellt: llama-3.2-3b-qnn)
  ├─ InferenceEngine (generálás: stream/non-stream)
  ├─ ConversationManager (history, max_history=10)
  ├─ AnalyticsTracker (GPT-4 fallback? analytics)
  └─ Security (rate limit: slowapi, API key auth)

Endpointek:
  GET  /health
  POST /chat (streaming SSE, 20/min rate limit, API key kell)
  DELETE /conversation (history clear)
  POST /admin/cleanup-old-data (GDPR placeholder)
```

**NPU?** IGEN — `config/model_config.json`: `"model_path": "./models/llama-3.2-3b-qnn"` (QNN modell, Qualcomm NPU).

---

## 3. GenieX átjárhatóság

**EZ MÁR NPU (QNN), de NEM GenieX!** A modell: `llama-3.2-3b-qnn` (QNN format, de nem GenieX szolgáltatás).
Két lehetőség:
- **A)** Saját QNN modell betöltés (onnxruntime-qnn vagy hasonló) — nem GenieX :18181
- **B)** GenieX-re váltás: a `model_loader.py` átírása, hogy GenieX Qwen3-4B :18181-et hívjon OpenAI SDK-val

**Jelenleg:** saját QNN inferencia (llama-3.2-3b). **GenieX átjárhatóság: KÖZEPES** — az InferenceEngine átírása szükséges (OpenAI SDK → GenieX base_url), de az API struktúra (FastAPI, /chat, streaming) kompatibilis.

---

## 4. Tech debt (VALÓS BUG-OK!)

1. **🔴 `customer_service_agent.py` ÜRES (0 byte)!** — Ez a repo központi fájlja, de ÜRES. Hiányzik a customer service logika. KRITIKUS.
2. **🔴 `inference_engine.py`, `model_loader.py`, `security.py` NINCS A KLÓNBAN?** — az api_server.py importálja őket (`from .inference_engine import InferenceEngine`), de a `src/` mappában csak: agent_framework, api_server, conversation_manager, customer_service_agent (üres), document_loader, analytics_tracker. Hiányoznak: inference_engine, model_loader, security!
3. **API key default**: `os.environ.get("API_KEY", "your-secret-api-key")` — ha nincs env, a default "your-secret-api-key" aktív. Biztonsági kockázat (bárki bejut).
4. **Nincs README** — csak kód, dokumentáció hiányzik
5. **Hardcoded path**: `./models/llama-3.2-3b-qnn` — relatív, más gépen nem található

---

## 5. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| API key auth | Van (X-API-Key header) | ⚠️ Default "your-secret-api-key" ha nincs env! |
| Rate limit | slowapi (20/min) | ✅ Jó |
| CORS | Nincs említve | ⚠️ Hiányzik (más frontend nem éri el) |
| Port | 127.0.0.1 | ✅ Helyi |

---

## 6. Összegzés

**Állapot:** Érdekes API skeleton, de **HIÁNYOS** — ÜRES customer_service_agent.py + hiányzó inference_engine/model_loader/security fájlok. Nem futtatható jelenleg (import error).
**GenieX reláció:** Saját QNN (llama-3.2-3b), NEM GenieX. Átjárhatóság közepes (InferenceEngine átírása kell).
**Tech debt:** 🔴 KRITIKUS — hiányzó fájlok (customer_service_agent üres, inference_engine/model_loader/security nincs a klónban). Ezt a repót **ÚJRA KELL ÉPÍTENI** vagy klónozni a hiányzó fájlokkal.
**Döntés:** A `gaiagent-ai-stack` főrepo README-jében: snapdragon_AI = "WIP, hiányzó fájlok, nem futtatható" — vagy pótolni kell a fájlokat (de az már kódírás, nem csak audit).

**Fájlok olvasva:** main.py, api_server.py, model_config.json, customer_service_agent.py (üres!), src/ structure
**Nem olvastam:** inference_engine.py (HIÁNYZIK a klónban!), model_loader.py (HIÁNYZIK!)
