# ai-core — Pan Audit Jelentés

**Dátum:** 2026-08-18 · **Auditor:** Pan (MiMo 2.5) · **Állapot:** ⏳ OLVASÁS + JELENTÉS (NE módosítsd!)

---

## 1. Mi ez?

Egységes AI backend orkesztrátor a vivo2 (Snapdragon X Elite) gépen.
- **Funkció:** "Egyetlen front-door" minden lokális és cloud LLM-hez
- **Kliensek:** meetcore, hu-voice-assistant, bármilyen OpenAI SDK kliens
- **Fő komponens:** LiteLLM proxy (WSL, :4000) — single API, fallback chain
- **Beépített szolgáltatások:** Ollama, GenieAPIService, GenieX, LiteLLM

---

## 2. Architektúra

```
meetcore / hu-voice-assistant / OpenAI kliens
         ↓
    LiteLLM proxy (WSL, :4000)
         ↓ fallback chain
┌────────────┬──────────┬──────────┬───────┬────────┐
│ genie-npu  │ nexa-npu │ ollama   │ groq  │ gemini │
│ :8912 (Win)│ :18181   │ :11434   │ cloud │ cloud  │
└────────────┴──────────┴──────────┴───────┴────────┘
```

**Forrás:** `config/litellm.yaml`

---

## 3. GenieX átjárhatóság

### ✅ GenieX MÁR BEÉPÍTVE:

| Komponens | Állapot | Részletek |
|-----------|---------|-----------|
| `services/geniex.ps1` | ✅ Kész | GenieX SDK indítása Windows-on, port 18181 |
| `ai-core.ps1` | ✅ Beépítve | `$Services` listában: `geniex` szolgáltatás, port 18181 |
| `config/litellm.yaml` | ⚠️ RÉSZLEGES | `nexa-local` és `default` modellek NexaAI prefix-szel |
| `config/.env.example` | ✅ Variábil | `GENIEX_PORT`, `GENIEX_DATADIR`, `GENIEX_DEFAULT_MODEL` |

### ⚠️ PROBLÉMÁK:

1. **`litellm.yaml` NexaAI prefix:**
   - `model: openai/NexaAI/qwen3-4B-npu` — ez a Nexa SDK modell neve
   - GenieX más modell nevet ad vissza (`qualcomm/Qwen3-4B-Instruct-2507`)
   - **KONFLIKTUS:** ha GenieX fut :18181-en, a litellm `NexaAI/qwen3-4B-npu` keresi, de GenieX más ID-t ad

2. **`config/litellm.yaml` nincs `geniex-local` entry:**
   - Van `nexa-local` (port :18181) és `genie-npu` (port :8912)
   - De **nincs külön `geniex-local`** entry a GenieX számára
   - A `nexa-local` használja :18181-et — ha GenieX fut, a Nexa model name nem fog stimmelni

3. **Fallback chain:**
   - `default: [nexa-local, ollama-qwen, groq-llama, ...]`
   - Ha a `nexa-local` nem működik (GenieX más model name), a chain továbblép
   - De a GenieX NPU gyorsítás kihasználatlan maradhat

---

## 4. GenieX konzisztencia a meetcore-val

| Szint | ai-core | meetcore | Konzisztens? |
|-------|---------|----------|-------------|
| **Port** | 18181 (geniex.ps1) | 18181 | ✅ IGEN |
| **Exe** | `geniex serve` | `geniex serve` | ✅ IGEN |
| **Modell** | `NexaAI/qwen3-4B-npu` (litellm) | `qualcomm/Qwen3-4B-Instruct-2507` | ⚠️ NEM — különböző model name! |
| **API** | OpenAI-kompat /v1 | OpenAI-kompat /v1 | ✅ IGEN |

**Összegzés:** A port és exe konzisztens, de a **modell név különbözik** — ez a litellm.yaml-ban okozhat problémát.

---

## 5. Legacy referenciák (OLVASÁS, nem módosítás!)

| Fájl | Legacy | Megjegyzés |
|------|--------|-----------|
| `ai-core.ps1:9` | `GenieAPIService (NPU, runtime-detected port)` | Komment — leírás |
| `ai-core.ps1:190` | `Get-RunningProcess -Name 'GenieAPIService'` | Process detection — legacy |
| `services/genie.ps1` | GenieAPIService szolgáltatás | Legacy szolgáltatás — megtartva |
| `services/nexa.ps1` | Nexa SDK szolgáltatás | Legacy szolgáltatás — megtartva |
| `config/litellm.yaml:16-17,20,74` | `NexaAI/qwen3-4B-npu` | Model name —可能 konfliktus GenieX-szel |
| `config/genie_config.template.json` | GenieAPIService config | Legacy template |
| `docs/architecture/ai-core-architecture.md:50` | `GenieAPIService (:8912) — replaced by GenieX` | Dokumentáció — történelmi magyarázat |

---

## 6. Tech debt

| Tétel | Prioritás | Megjegyzés |
|-------|-----------|-----------|
| `litellm.yaml` NexaAI prefix | 🔴 MAGAS | GenieX model name konfliktus |
| Nincs `geniex-local` litellm entry | 🔴 MAGAS | GenieX nincs a fallback chain-ben |
| `services/genie.ps1` | 🟡 KÖZEPES | Legacy szolgáltatás, visszafelé kompatibilitás |
| `services/nexa.ps1` | 🟡 KÖZEPES | Legacy szolgáltatás |
| `config/genie_config.template.json` | 🟢 ALACSONY | Template, nem futó kód |
| WSL függés | 🟡 KÖZEPES | LiteLLM WSL-ben fut |

---

## 7. Összegzés

**ai-core:**
- ✅ **GenieX beépítve** (`geniex.ps1`, `ai-core.ps1` szolgáltatás listában)
- ⚠️ **litellm.yaml konfliktus** — `NexaAI/qwen3-4B-npu` model name nem stimmel a GenieX-szel
- ⚠️ **Nincs külön GenieX litellm entry** — a `nexa-local` használja :18181-et
- ✅ **Port konzisztens** — :18181 mindkét projektnél
- ✅ **Exe konzisztens** — `geniex serve` mindkét projektnél

**JAVASLAT (nem módosítás, csak jelentés):**
1. `litellm.yaml`-ban `nexa-local` → `geniex-local`, model name: `openai/qualcomm/Qwen3-4B-Instruct-2507`
2. Vagy: GenieX indításakor a `geniex list` visszaadja a tényleges model ID-t, azt kell a litellm.yaml-ban használni

**Konzisztens a meetcore-val?** RÉSZLEGESEN — port/exe igen, modell name nem.
