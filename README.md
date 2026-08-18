# gaiagent-ai-stack

**gaiagent0 AI stack — főrepo.** Minden AI/NPU projekt auditálva, strukturálva, GenieX migráció nyomon követve.

> **"Alkalmazhatóként" értelmezés:** A repók Docker/WSL most nincs telepítve a gépen. A dokumentációk, architektúra leírások, port térképek és konfigok **időállóak** — ha később lesz WSL/Docker környezet, újrahasználhatók. A futtató scriptek (`.ps1`, `.bat`, `docker-compose`) csak akkor érvényesek, ha az adott környezet jelen van.

---

## 🏗️ Architektúra (3 réteg, vivo2 gép)

```
Windows ARM64 (AI Compute Layer) — NPU-first
├── GenieX           :18181  NPU inference (Qwen3-4B-Instruct)   ← A CÉL STACK
├── Foundry Local    :5272  QNN inference (DeepSeek-R1-7B)      ← Alternatív NPU
├── Ollama           :11434 CPU inference (20+ modell)
├── MTP servers      :8081/:8083 llama.cpp (nagy modellek, CPU)
├── Open WebUI       :8091  Chat frontend
├── Hermes GW        :8642  Agent runtime
└── control-center   :5757  AI service dashboard

WSL2 Ubuntu 24.04 ARM64 (Infrastructure Layer)
├── LiteLLM Proxy    :4001  AI gateway + UI
├── ChromaDB         :8001  Vector memory
└── Redis            :6379  Session cache

CT305 pve-03 Docker (Always-On Layer)
├── PostgreSQL :5432 · Qdrant :6333 · n8n :5678 · SearXNG :8080 · Mem0 :8888
```

---

## 📦 Alrepók (submodule-ok)

| Alrepo | Típus | Szerep | GenieX status |
|---|---|---|---|
| [meetcore](https://github.com/gaiagent0/meetcore) | publikus | Meeting assistant (NPU-first) | ✅ **MIGRÁLT** (Qwen3-4B) |
| [hu-voice-assistant](https://github.com/gaiagent0/hu-voice-assistant) | privát | Hangsegéd (Piper+F5, Whisper, LiteLLM) | ⚠️ Részleges (LLM könnyű) |
| [hu-ai-chat](https://github.com/gaiagent0/hu-ai-chat) | publikus | Cloud Qwen chat | N/A (cloud) |
| [meetily-snapdragon](https://github.com/gaiagent0/meetily-snapdragon) | privát | meetcore előd | ❌ Redundáns (archívum) |
| [ai-core](https://github.com/gaiagent0/ai-core) | publikus | LiteLLM proxy + NPU orchestration | ✅ GenieX beépítve (ps1) |
| [control-center](https://github.com/gaiagent0/control-center) | privát | AI service dashboard | ✅ **MIGRÁLT** (Pan, :18181) |
| [vivo2-ai-stack-2026](https://github.com/gaiagent0/vivo2-ai-stack-2026) | privát | Infra docs + konfig | ✅ GenieX (Qwen3-4B) |
| [snapdragon_AI](https://github.com/gaiagent0/snapdragon_AI) | privát | Python agent framework | ⚠️ Saját QNN (llama-3.2-3b) |
| [hermes-foundry-npu](https://github.com/gaiagent0/hermes-foundry-npu) | publikus | Hermes + Foundry NPU | N/A (Foundry, nem GenieX) |
| [llama-mtp-snapdragon](https://github.com/gaiagent0/llama-mtp-snapdragon) | publikus | llama.cpp MTP build | N/A (CPU llama.cpp) |
| [snapdragon-ai-stack](https://github.com/gaiagent0/snapdragon-ai-stack) | publikus | AI stack docs (legacy) | ❌ GenieAPIService (archívum) |
| [ai-landing](https://github.com/gaiagent0/ai-landing) | publikus | Landing page (Next.js) | N/A (marketing) |

---

## 🔍 Audit jelentések

Mappa: [`audit/`](audit/)

| Jelentés | Repó | Szerző |
|---|---|---|
| [meetcore-backend-REVIEW.md](audit/meetcore-backend-REVIEW.md) | meetcore | Bot C (solar4) |
| [hu-ai-chat-AUDIT.md](audit/hu-ai-chat-AUDIT.md) | hu-ai-chat | Hermes |
| [hu-voice-assistant-AUDIT.md](audit/hu-voice-assistant-AUDIT.md) | hu-voice-assistant | Hermes (Bot D helyett) |
| [hu-voice-assistant-PAN-AUDIT.md](audit/hu-voice-assistant-PAN-AUDIT.md) | hu-voice-assistant | Pan (MiMo 2.5) |
| [meetily-snapdragon-AUDIT.md](audit/meetily-snapdragon-AUDIT.md) | meetily-snapdragon | Hermes (Bot E helyett) |
| [vivo2-ai-stack-2026-AUDIT.md](audit/vivo2-ai-stack-2026-AUDIT.md) | vivo2-ai-stack-2026 | Hermes (Bot F helyett) |
| [snapdragon_AI-AUDIT.md](audit/snapdragon_AI-AUDIT.md) | snapdragon_AI | Hermes (Bot F helyett) |
| [hermes-foundry-npu-AUDIT.md](audit/hermes-foundry-npu-AUDIT.md) | hermes-foundry-npu | Hermes (Bot G helyett) |
| [llama-mtp-snapdragon-AUDIT.md](audit/llama-mtp-snapdragon-AUDIT.md) | llama-mtp-snapdragon | Hermes (Bot G helyett) |
| [snapdragon-ai-stack-AUDIT.md](audit/snapdragon-ai-stack-AUDIT.md) | snapdragon-ai-stack | Hermes (Bot H helyett) |
| [ai-landing-AUDIT.md](audit/ai-landing-AUDIT.md) | ai-landing | Hermes (Bot H helyett) |
| [control-center-AUDIT.md](audit/control-center-AUDIT.md) | control-center | Pan (MiMo 2.5) |
| [ai-core-AUDIT.md](audit/ai-core-AUDIT.md) | ai-core | Pan (MiMo 2.5) |
| [REPO-STRUCTURE.md](REPO-STRUCTURE.md) | — | Hermes (struktúra terv) |

---

## 🚨 Kritikus megállapítások

1. **🔴 snapdragon_AI**: Hiányzó fájlok (`customer_service_agent.py` üres, `inference_engine.py`/`model_loader.py`/`security.py` nincs a klónban) → **nem futtatható**
2. **🔴 ai-landing**: Contact form letiltva (Turnstile hiba) → **kapcsolatfelvétel nem működik**
3. **⚠️ ai-core**: `litellm.yaml`-ban NexaAI prefix konfliktus a model name-nél (GenieX migráció részleges)
4. **⚠️ snapdragon-ai-stack**: Elavult (GenieAPIService :8912 legacy) → archívum, a vivo2-ai-stack-2026 a referencia
5. **✅ control-center**: GenieX-re migrálva (Pan), konzisztens a meetcore-rel
6. **✅ vivo2-ai-stack-2026**: Már GenieX Qwen3-4B (konzisztens)

---

## 📋 Struktúra terv

Részletesen: [REPO-STRUCTURE.md](REPO-STRUCTURE.md)
- **Főrepo** (ez) = bejárati pont + audit dokumentáció
- **Alrepók** = git submodule (publikus + privát)
- **"Alkalmazhatóként"** = Docker/WSL nélkül is érvényes elvek

---

*Audit: 2026-08-18 · Hermes (orchestrator) + Pan (MiMo 2.5) + solar4 Bots (F/G/H) · GenieX migráció nyomon követve*
