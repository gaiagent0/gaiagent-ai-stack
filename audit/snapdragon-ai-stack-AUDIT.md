# snapdragon-ai-stack — Audit

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\snapdragon-ai-stack`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Verzió:** 2026-03-30 · **Státusz:** ELAVULT (GenieAPIService era)
**Összefüggés:** gaiagent0 repócsalád auditja — Snapdragon AI stack docs

---

## 1. Mi ez?

**Complete local AI stack for Snapdragon X Elite (ARM64 Windows)** — dokumentáció repo.
NEM futtatható kód, hanem **reference architecture** (GenieAPIService era, 2026-03).
"Alkalmazhatóként" értelmezendő: Docker/WSL most nincs, de a docs újrahasználhatók ha lesz.

---

## 2. Architektúra

```
Windows ARM64 (Native)
├── GenieAPIService   :8912  NPU inference (Llama 3.1 8B, QNN 2.38)   ← LEGACY!
├── Ollama            :11434 CPU inference (GGUF, NEON SIMD)
└── Open WebUI        :8080  Chat frontend (uvx)

WSL2 Ubuntu 24.04 ARM64
├── LiteLLM Proxy     :4000  Unified API
├── ChromaDB          :8001  Vector store (RAG)
├── n8n               :5678  Workflow automation
└── SearXNG           :8080  Local web search
```

**GenieAPIService vs GenieX:** EZ még a RÉGI `GenieAPIService :8912` (Llama 3.1 8B). A meetcore már GenieX Qwen3-4B :18181-re váltott. **ELAVULT!**

---

## 3. GenieX átjárhatóság

**EZ NEM GENIEX — LEGACY GenieAPIService!**
- `:8912` GenieAPIService = Llama 3.1 8B (régi QNN 2.38)
- A meetcore / vivo2-ai-stack-2026 már GenieX Qwen3-4B :18181

**REDUNDANCIA:** Igen — a `vivo2-ai-stack-2026` (2026-06) az ÚJABB verzió. Ez a repo (2026-03) a **régebbi, elavult** dokumentáció.
**Döntés:** Ez a repo **ARCHÍVUM** — a vivo2-ai-stack-2026 a referencia most.

---

## 4. Tech debt

1. **🔴 GenieAPIService legacy mindenhol** — `:8912`, Llama 3.1 8B, QNN 2.38. A GenieX migráció nem történt meg ebben a repóban
2. **NPU driver 24H2 függés** — "Earlier builds have no NPU support" — korlátozó
3. **Open WebUI pip hiba ARM64** — "No ARM64 wheel" → uvx fallback (dokumentálva, de nem ideális)
4. **Port ütközés**: Open WebUI :8080 (WSL2) vs SearXNG :8080 (WSL2) — ugyanaz a port két szolgáltatásnál!
5. **Hardcoded path-ek**: `C:\AI\models\ollama`, `C:\AI\GenieAPIService_cpp`, `C:\AI\openwebui`

---

## 5. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| Binding | 127.0.0.1 alapértelmezett | ✅ Jó (netstat check dokumentálva) |
| API key | Nincs említve | ⚠️ Feltételezhetően .env |
| Firewall | autostart script említve | ✅ Dokumentálva |

---

## 6. Összegzés

**Állapot:** Érett dokumentáció, de **ELAVULT** (GenieAPIService era, 2026-03).
**GenieX reláció:** ❌ NEM GenieX — legacy GenieAPIService :8912. A vivo2-ai-stack-2026 a friss.
**Tech debt:** GenieAPIService legacy, port ütközés (:8080 dupla), 24H2 függés.
**Döntés:** **ARCHÍVUM** — a `gaiagent-ai-stack` főrepo README-jében "legacy docs"ként hivatkozandó, a vivo2-ai-stack-2026 a referencia.

**Fájlok olvasva:** README.md (GitHub adatok + klón structure), docs/* (structure)
**Nem olvastam mélyebben:** docs/*.md (a README fedte a stack-et)
