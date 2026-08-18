# vivo2-ai-stack-2026 — Audit

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\vivo2-ai-stack-2026`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Verzió:** 2026-06-15 · **Státusz:** Aktív dokumentáció
**Összefüggés:** gaiagent0 repócsalád auditja — vivo2 gép AI OS

---

## 1. Mi ez?

**Snapdragon X Elite Hybrid AI Operating System** — infrastruktúra dokumentáció + konfigurációk.
NEM futtatható kód, hanem **reference architecture**: Windows ARM64 + WSL2 + CT305 Proxmox stack leírása.
"Alkalmazhatóként" értelmezendő: Docker/WSL most nincs, de a konfigok (docker-compose, litellm_config) újrahasználhatók ha lesz ilyen környezet.

---

## 2. Architektúra (3 réteg)

```
Windows ARM64 (AI Compute Layer)
├── GenieAPIService   :8912  NPU inference (Qwen3-4B W4A16)   ← GENIEX!
├── Foundry Local     :5272  QNN inference (DeepSeek-R1-7B)
├── Ollama            :11434 CPU inference (20+ modell)
├── Open WebUI        :8091  Chat frontend
├── Hermes GW         :8642  Agent runtime
├── vivo-embed        :7272  Embedding pipeline
└── MTP servers       :8081/:8083  llama.cpp (kézi)

WSL2 Ubuntu 24.04 ARM64 (Infrastructure Layer)
├── LiteLLM Docker    :4001  AI gateway + UI
├── ChromaDB Docker   :8001  Vector memory
└── Redis Docker      :6379  Session cache

CT305 pve-03 Docker (Always-On Layer)
├── PostgreSQL        :5432  Structured memory + LiteLLM DB
├── Qdrant            :6333  Vector store
├── n8n               :5678  Automation
├── SearXNG           :8080  Private search
└── Mem0              :8888  Semantic memory
```

**Rétegek konzisztensek?** Igen — tiszta elválasztás: Windows (NPU) / WSL2 (infra) / Proxmox (always-on).

---

## 3. GenieX átjárhatóság

**EZ MÁR GENIEX!** A README explicit írja: `GenieAPIService :8912  NPU inference (Qwen3-4B W4A16)`.
Ez a **legegységesebb GenieX említés** a repók közül — a meetcore is Qwen3-4B-re váltott, itt is az van.
A `vivo-embed` (7272) is NPU-s embedding pipeline.

**Konzisztencia a meetcore-rel:** ✅ Igen — mindkettő GenieX Qwen3-4B. Ez a repo a "nagy kép" (orchestration), a meetcore a "alkalmazás" (meeting assistant).

---

## 4. Tech debt

1. **GenieAPIService vs GenieX névkeveredés**: a port :8912 "GenieAPIService" néven fut, de a modell Qwen3-4B (GenieX). A név legacy (régen llama3.1-8b volt), most már GenieX.
2. **Hardcoded path-ek**: `E:\models`, `D:\` (path-fix log), `C:\AI\` — más gépen nem működik
3. **Docker/WSL függés**: a WSL2 + CT305 rétegek most NINCSENEK (Docker nélkül) — a docs "alkalmazhatóként" érvényesek
4. **Port ütközés veszély**: Open WebUI :8091 (vivo2) vs :8080 (snapdragon-ai-stack docs) — két külön repo, eltérő port
5. **Hiányzó dokumentáció**: a `configs/` csak template-ek, nincs "hogyan állítsd fel" lépésről lépésre

---

## 5. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| API key/.env | Nincs említve a README-ben | ⚠️ Feltételezhetően .env-ben (litellm_config.yaml) |
| CORS | N/A (doksi repo) | ⚠️ A LiteLLM proxy-nál fontos lenne |
| Port expozíció | 127.0.0.1 alapértelmezett? | ⚠️ Nincs dokumentálva |

---

## 6. Összegzés

**Állapot:** Érett infrastruktúra dokumentáció, GenieX-kompatibilis (Qwen3-4B).
**GenieX reláció:** ✅ Konzisztens a meetcore-rel (mindkettő GenieX Qwen3-4B).
**Tech debt:** GenieAPIService vs GenieX név, hardcoded path-ek, Docker/WSL függés (most nincs).
**Döntés:** A `gaiagent-ai-stack` főrepo **fő dokumentációja** lehet — ez írja le a teljes vivo2 AI OS-t. A meetcore/hu-voice-assistant ehhez kapcsolódik.

**Fájlok olvasva:** README.md, docs/* (GitHub adatok + klón structure)
**Nem olvastam mélyebben:** configs/*.yml (template-ek, a README fedte)
