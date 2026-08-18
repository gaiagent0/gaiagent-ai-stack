# gaiagent0 AI repók — Audit + Strukturálási javaslat

**Dátum:** 2026-08-18 · **Cél:** Minden AI-témájú gaiagent0 repo auditálása, átstrukturálási javaslat (főrepo + alrepok), "alkalmazhatóként" értelmezés (Docker/WSL most nincs, de ha lesz, érvényes).

---

## 1. Kutatás — Legjobb gyakorlat (internet, 2025-2026)

Források: aviator.co, augmentcode.com, circleci.com, medium.com (raminmammadzada), reddit.com/r/git

**Monorepo vs Polyrepo döntési fa:**
- **Monorepo**: kis-közepes csapat (1-10 fő), szorosan kapcsolódó szolgáltatások, közös függőségek, **AI coding assistant egységes kontextus igénye** → ERŐSEN AJÁNLOTT
- **Polyrepo**: független release ciklus, szigorú access control (pl. vállalati titok), külön csapatok
- **Git submodules**: közös kód verzió-pinning-gal, de bonyolult (`git submodule update` szükséges, rossz hírű)
- **Git subtree**: submodule alternatíva, de history bonyolult
- **Google repo tool**: sok kis repo (Android-style) — 1 embernek túlzás

**Kulcs tanulság (AI era):** *"If your team uses AI coding assistants, a monorepo provides the unified context these tools need to be most effective."* — CircleCI. Mivel Te (István) Hermes-szel + Pan-nel + Bots-okal dolgozol, a **monorepo/uni-repo a legjobb**, mert:
1. Hermes egy helyen látja az összes kontextust (keresés, refactor, cross-repo változások)
2. Egységes CI/CD, dependency management
3. Atomikus változások (pl. GenieX migráció minden repóban 1 commit)

**DE:** a privát repók (control-center, vivo2-ai-stack-2026, snapdragon_AI, hu-voice-assistant, meetily-snapdragon) érzékenyebbek lehetnek, és a publikusak (meetcore, ai-core, snapdragon-ai-stack) mások is láthatják.

→ **HIBRID MEGOLDÁS: "FŐREPO + ALREPOK" (meta-repo monorepo hybrid)**

---

## 2. Javasolt struktúra — `gaiagent-ai-stack` (új főrepo)

### FŐREPO: `gaiagent-ai-stack` (publikus, új)
- **Bejárati pont**: README a teljes AI stack architektúrával (vivo2 + snapdragon + CT305 Proxmox)
- **Alrepok**: git submodule-ként hivatkozza a meglévő repókat
- **Előny**: `git clone --recurse-submodules` letölt mindent; Hermes látja az összes context-et; privát repók privátak maradnak (submodule URL = privát, csak a token-tel rendelkezőknek jön le)

### ALREPOK (meglévők, submodule-ként):
| Alrepo | Típus | Szerep | Állapot |
|---|---|---|---|
| `meetcore` | publikus | Meeting assistant (NPU-first, GenieX migrált) | ✅ Auditálva, javítva |
| `hu-voice-assistant` | **privát** | Hangsegéd platform (Piper+F5, Whisper, LiteLLM) | ✅ Auditálva |
| `hu-ai-chat` | publikus | Cloud Qwen chat | ✅ Auditálva |
| `meetily-snapdragon` | **privát** | meetcore előd (redundáns) | ✅ Auditálva, archívum |
| `ai-core` | publikus | LiteLLM proxy + NPU orchestration (vivo2) | ⏳ Auditálandó |
| `control-center` | **privát** | AI service dashboard (indít/leállít/monitor) | ⏳ Auditálandó |
| `vivo2-ai-stack-2026` | **privát** | Infra dokumentáció + konfig (Docker/WSL2/Proxmox) | ⏳ Auditálandó |
| `snapdragon_AI` | **privát** | Python agent framework + RAG | ⏳ Auditálandó |
| `hermes-foundry-npu` | publikus | Hermes + Foundry NPU integráció | ⏳ Auditálandó |
| `llama-mtp-snapdragon` | publikus | llama.cpp MTP build (ARM64) | ⏳ Auditálandó |
| `snapdragon-ai-stack` | publikus | Snapdragon AI stack docs (GenieAPIService era) | ⏳ Auditálandó |
| `ai-landing` | publikus | Landing page (Next.js) | ⏳ Auditálandó |

### Mappa struktúra a főrepoban:
```
gaiagent-ai-stack/
├── README.md                    # Teljes stack áttekintés
├── docs/
│   ├── ARCHITECTURE.md          # vivo2 + snapdragon + CT305 topológia
│   ├── REPO-MAP.md              # Alrepok listája + szerepük
│   └── STACK-CURRENT.md         # "Alkalmazhatóként" — jelenlegi állapot (Docker/WSL nélkül is érvényes elvek)
├── apps/                        # submodule-ok (publikus)
│   ├── meetcore/
│   ├── hu-ai-chat/
│   ├── ai-core/
│   ├── hermes-foundry-npu/
│   ├── llama-mtp-snapdragon/
│   ├── snapdragon-ai-stack/
│   └── ai-landing/
├── private/                     # submodule-ok (privát, csak token-nel)
│   ├── hu-voice-assistant/
│   ├── meetily-snapdragon/
│   ├── control-center/
│   ├── vivo2-ai-stack-2026/
│   └── snapdragon_AI/
├── audit/                       # audit jelentések (ez a mappa!)
│   ├── meetcore-backend-REVIEW.md
│   ├── hu-ai-chat-AUDIT.md
│   ├── hu-voice-assistant-AUDIT.md
│   ├── hu-voice-assistant-PAN-AUDIT.md
│   ├── meetily-snapdragon-AUDIT.md
│   └── REPO-STRUCTURE.md        # ez a fájl
└── .gitmodules                 # submodule konfig
```

---

## 3. "Alkalmazhatóként" értelmezés (Docker/WSL nélkül is)

Mivel **Docker és WSL most nincs**, a repók tartalmát **architektúrális/elv szinten** kell értelmezni:
- A `vivo2-ai-stack-2026` és `snapdragon-ai-stack` **dokumentáció**, nem futtatható kód → ezek **reference architecture**-ként érvényesek (ha később lesz WSL/Docker, a konfigok újra használhatók)
- A `control-center`, `ai-core`, `meetcore`, `hu-voice-assistant` **futtatható kód**, de a jelenlegi gépen (Windows ARM64, NPU, de WSL/Docker nélkül) csak a **Windows-native** részek érvényesek (GenieAPIService, Ollama, MTP)
- A `snapdragon_AI` Python agent framework — `pip install` + `python main.py` működik WSL/Docker nélkül is (ha a függőségek települnek)

**Következtetés:** A repók **"alkalmazhatóként" (as-applicable)** értendők — a README/benchmark adatok, port térképek, architektúra leírások **időállóak**, a futtató scriptek (`.bat`, `.ps1`, `docker-compose`) csak akkor érvényesek, ha az adott környezet (WSL/Docker) jelen van.

---

## 4. Repó állapot összesítés (auditáltak)

| Repó | Állapot | GenieX migráció | Megjegyzés |
|---|---|---|---|
| meetcore | ✅ Kész | ✅ MEGTÖRTÉNT | 74 passed, TTS+live-ASR+RAG bent |
| hu-ai-chat | ✅ Kész | N/A (cloud) | Reference only |
| hu-voice-assistant | ✅ Kész | ⚠️ RÉSZLEGES | LLM könnyű, ASR nem, TTS részleges |
| meetily-snapdragon | ✅ Kész | N/A | Redundáns, archívum |
| ai-core | ⏳ Következő | ⚠️ nexa→geniex említve | LiteLLM proxy vivo2-nak |
| control-center | ⏳ Következő | ❌ GenieAPIService (legacy) | Kritikus dashboard |
| vivo2-ai-stack-2026 | ⏳ Következő | ✅ GenieX említve | Infra docs |
| snapdragon_AI | ⏳ Következő | ❓ Ismeretlen | Agent framework |
| hermes-foundry-npu | ⏳ Következő | N/A (Foundry) | Hermes+NPU |
| llama-mtp-snapdragon | ⏳ Következő | N/A (llama.cpp) | MTP build |
| snapdragon-ai-stack | ⏳ Következő | ❌ GenieAPIService (legacy) | Régi docs |
| ai-landing | ⏳ Következő | N/A | Landing page |

---

## 5. Akcióterv

1. **FŐREPO létrehozása**: `gaiagent-ai-stack` (publikus) GitHub-on
2. **Submodule-ok hozzáadása**: a 12 alrepo (publikus + privát)
3. **Dokumentáció**: README + docs/ (ARCHITECTURE, REPO-MAP, STACK-CURRENT)
4. **Audit mappa**: ide kerülnek a jelentések
5. **Push**: főrepo + submodule-ok (privátak csak ha Te akarod)
6. **További repók auditja**: ai-core, control-center, vivo2, snapdragon_AI, hermes-foundry-npu, llama-mtp, snapdragon-ai-stack, ai-landing (Botok + Pan)

---

*Kutatás: internet (2025-2026 best practices) + 11 repo klón audit (2026-08-18)*
*Hermes orchestrator + Pan (MiMo 2.5) + solar4 Bots koordinációval*
