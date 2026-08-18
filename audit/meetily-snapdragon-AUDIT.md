# meetily-snapdragon — Audit + Redundancia elemzés

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\meetily-snapdragon`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Összefüggés:** gaiagent0 repócsalád auditja — meetcore elődje

---

## 1. Mi a meetily-snapdragon?

A **meetcore repo korábbi ága/snapshot-ja** (2026-04-13, a meetcore klón 2026-04-29 előtt).
A CLAUDE.md maga is ezt írja: *"Ez a `meetcore` repo Snapdragon ága"*.
Ugyanaz az architektúra: FastAPI :5167 + Next.js 15 :3118/hu + next-intl (hu/en) + SQLite.

**Mit tud a meetcore NEM?** → Semmit. A meetcore a *fejlettebb* változat.
A meetily csak historikus érdekesség (a meetcore fejlődési állapota 2 héttel a meetcore klón előtt).

---

## 2. Architektúra (2026-04-13 állapot)

| Réteg | meetily-snapdragon | meetcore (klón, 2026-08-18) |
|---|---|---|
| **ASR** | NexaAI Parakeet TDT 0.6B v3 (NPU, :18181) | **GenieX-stack Whisper-Base QNN** (Hexagon NPU) |
| **LLM** | GenieAPIService llama3.1-8b (OFFLINE ❌) / NexaAI Llama3.2-3B | **GenieX Qwen3-4B** (:18181, működik) |
| **TTS** | ❌ NINCS | ✅ Piper hu_HU-anna + F5-TTS voice-clone |
| **Live ASR** | ❌ NINCS | ✅ WebSocket (/ws/live-asr) |
| **RAG** | ❌ NINCS | ✅ ChromaDB |
| **Default provider** | ollama | geniex (Qwen3-4B) |

**Következtetés:** a meetily-snapdragon funkcionálisan a meetcore *részhalmaza* — kevesebbet tud, elavultabb futtatókkal (NexaAI Parakeet + offline GenieAPIService).

---

## 3. REDUNDANCIA elemzés

**A meetily-snapdragon TELJESEN REDUNDÁNS a meetcore mellett.**

- A meetcore klón (GenieX migrált, TTS+F5+live-ASR+RAG bent) **tartalmazza és felülmúlja** a meetily-t.
- A meetily-ben nincs olyan funkció, ami a meetcore-ban ne lenne meg (vagy ne lenne jobb).
- A meetily csak a **fejlesztési történet** része — referenciaként érdekes (hogyan jutott el a NexaAI Parakeet + offline GenieAPIService állapotból a GenieX Qwen3-4B + Whisper-Base QNN állapotig).

**Döntés:** A meetily-snapdragon **nem kell** — beolvadt a meetcore-be. Karbantartási szempontból:
- Ha privát repo, hagyható archívumnak (nem törlendő, de nem fejlesztendő)
- Új funkciók a meetcore-ba kerüljenek

---

## 4. Tech debt (a meetily sajátja)

1. **ASR**: NexaAI Parakeet (legacy, `E:\models-nexa\...`) — GenieX Whisper-Base QNN váltotta fel
2. **LLM**: GenieAPIService llama3.1-8b **OFFLINE** (CLAUDE.md: ❌ offline) — a meetcoreban GenieX Qwen3-4B működik
3. **NINCS TTS** — a meetcoreban Piper+F5 van
4. **NINCS live ASR / RAG**
5. **Elavult függőségek**: README még GenieAPIService :8912, llama3.1-8b (a meetcore klónban már :18181 Qwen3-4B)
6. **CORS `*`** — production előtt szűkítendő (ugyanaz mint meetcore)

---

## 5. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| API key kezelés | in-app Settings UI → SQLite | ✅ Jó (meetcore-val megegyező) |
| CORS | `*` (dev) | ⚠️ Production előtt szűkítendő |
| `.env` / `*.db` | `.gitignore`-ban | ✅ Jó |

---

## 6. Összegzés

**Állapot:** A meetily-snapdragon a meetcore **elavult elődje** — funkcionálisan redundáns, elavult futtatókkal.
**Redundancia:** TELJES — a meetcore klón (GenieX, TTS, live-ASR, RAG) teljesen lefedi és felülmúlja.
**Mit érdemes átvinni:** Semmit — a meetcore már tartalmaz mindent, ami a meetily-ben volt, csak jobban.
**Döntés:** Archívumként tartható (privát repo), de **nem fejlesztendő** — az aktív fejlesztés a meetcore-ban történjen.

**Fájlok olvasva:** README.md, CLAUDE.md (architektúra, 2026-04-13 állapot)
**Nem olvastam mélyebben:** backend/app/*.py (a meetcore klónnal való azonosság miatt felesleges — ugyanaz az elavult kód)
