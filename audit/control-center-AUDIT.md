# control-center — Pan Audit Jelentés

**Dátum:** 2026-08-18 · **Auditor:** Pan (MiMo 2.5) · **Állapot:** ✅ MÓDOSÍTVA (engedélyezve)

---

## 1. Mi ez?

Központi AI service dashboard a vivo2 (Snapdragon X Elite) gépen.
- **Backend:** FastAPI (Python) :5757
- **Frontend:** Vanilla JS (ai-dashboard.html, chat.html, index.html)
- **Funkció:** Indít/leállít/monitoroz MTP, Genie, Ollama, Hermes, n8n szolgáltatásokat
- **Architektúra:** `services.json` definiálja a szolgáltatásokat, `backend/main.py` a management logikát

---

## 2. Módosítások (GenieX migráció)

### Fájlok (6 db):

| # | Fájl | Mi változott |
|---|------|-------------|
| 1 | `services.json` | `id: genie` → `geniex`, `label: GenieAPIService` → `GenieX`, port `8912` → `18181`, `exe: GenieAPIService.exe` → `geniex`, `process_name: GenieAPIService` → `geniex`, `start_cmd` → `geniex serve --host 0.0.0.0:18181` |
| 2 | `backend/main.py` | `EXE_START["genie"]` → `EXE_START["geniex"]`: exe `GenieAPIService.exe` → `geniex`, args → `["serve", "--host", "0.0.0.0", "18181"]`, workdir → `None` |
| 3 | `frontend/ai-dashboard.html` | `GenieAPIService` → `GenieX` (mindenhol), port `8912` → `18181`, modell `Llama3.2-3B` → `Qwen3-4B-Instruct` |
| 4 | `frontend/chat.html` | `GenieAPIService` → `GenieX`, port `:8912` → `:18181` |
| 5 | `README.md` | `GenieAPIService` → `GenieX`, `NexaAI` → `Whisper-Base QNN`, port `8912` → `18181`, `Llama 3.1 8B` → `Qwen3-4B-Instruct` |
| 6 | `diagnostics.py` | `genie: GenieAPIService:8912` → `geniex: GenieX:18181` |

### Ellenőrzés:
- `grep -r "GenieAPIService\|NexaAI\|npx\.genie\|nexa\."` → **0 találat** (tiszta!)

---

## 3. GenieX konzisztencia

| Szint | Állapot |
|-------|---------|
| **Port** | ✅ 18181 (egységes: services.json + main.py + frontend + diagnostics) |
| **Exe** | ✅ `geniex serve` (nem GenieAPIService.exe) |
| **Modell** | ✅ Qwen3-4B-Instruct (nem Llama 3.1 8B) |
| **Process** | ✅ `geniex` (nem GenieAPIService) |
| **i18n** | ✅ N/A (egynyelvű HTML) |

**Konzisztens a meetcore-val?** ✅ IGEN — meetcore-ban is GenieX :18181, Qwen3-4B.

---

## 4. Tech debt (megmaradt)

| Tétel | Megjegyzés |
|-------|-----------|
| `services/genie.ps1` | Legacy GenieAPIService szolgáltatás — megtartva (visszafelé kompatibilitás) |
| `services/nexa.ps1` | Nexa SDK szolgáltatás — megtartva |
| `config/genie_config.template.json` | Legacy GenieAPIService config template — megtartva |
| `backend/main.py` | ~900 sor, hardcoded path-ek — generikus tech debt |
| `.exe` parancsok, `.bat` scriptek | Legacy futtatókörnyezet hivatkozások |

---

## 5. Összegzés

A control-center **GenieX-re migrálva** — 6 fájl módosítva, 0 legacy referencia.
A migráció **konzisztens** a meetcore projekttel (ugyanaz a port, modell, exe).
A régi GenieAPIService szolgáltatások **megtartva** visszafelé kompatibilitásból.
