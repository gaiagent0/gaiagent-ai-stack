# MeetCore backend — GenieX migrációs kódfelülvizsgálat (REVIEW)

**Dátum:** 2026-08-18
**Reviewer:** Hermes subagent (kódfelülvizsgálat, csak olvasás)
**Forrás:** Stan subagent GenieX migrációja
**Skill-ek ellenőrzési alapja:** `geniex-npu` (v7.1.0), `arm64-voice-npu` (v0.1.0)
**Vizsgált fájlok:** `backend/app/whisper_npu.py`, `backend/app/transcript_processor.py` (geniex/npu provider rész), `backend/app/npu_routes.py`

---

## 1. Összefoglaló (verdict)

A migráció **nagyvonalakban konzisztens a skill-ekkel**: a GenieX provider (Qwen3-4B, `:18181`, OpenAI-kompatibilis) és a Whisper-Base QNN backend (QNNExecutionProvider `htp`, helyes forced-token sorrend, `no_think`/`<think:6124c78e>`-strip a #1294 miatt) a dokumentált tényeknek megfelelően van megírva. A skill-faktszintű hibák **nem** találhatók.

Ugyanakkor **3 valódi bug / inkonzisztencia** van, ebből 1 közepes (a `cpp` whisper.cpp fallback az API-n keresztül elérhetetlen + eltérő env-var név), 1 közepes (omnineural ág `NameError`), és 1 kisebb (stale `.env.example` / "GenieAPIService" elnevezés-maradványok a GenieX helyett). Ezek nem akadályozzák a fő útvonalat (npu/geniex + whisper-base-qnn), de a migráció minőségét rontják és a dokumentáció félrevezető.

**Összesítés:** Fő útvonal működőképes és skill-konzisztens; javítandó a backend-váltó logika, a dokumentáció, és 1 elszánt `NameError`.

---

## 2. Konzisztencia a skill-ekkel — ELLENŐRZÖTT PONTOK

### whisper_npu.py — Whisper-Base QNN (arm64-voice-npu skill)
| Ellenőrzendő | Skill tény | Kód állapota | OK? |
|---|---|---|---|
| QNN EP + htp | `providers=[("QNNExecutionProvider", {"backend_type":"htp"})]` (htp = Hexagon NPU) | L237: `providers=[("QNNExecutionProvider", {"backend_type": "htp"})]` | ✅ |
| Modell | `qualcomm/Whisper-Base` PRECOMPILED_QNN_ONNX, multilingual, hu out-of-box | L44-55: `qualcomm/Whisper-Base` zip, `language="hu"` (DEFAULT_LANG), encoder/decoder.onnx + tokenizer.json | ✅ |
| Forced token sorrend | standard Whisper multilingual decode | L356: `[sot, lang_hu, transcribe, no_timestamps]` = `<|startoftranscript|> <|hu|> <|transcribe|> <|notimestamps|>` | ✅ |
| Special token ID-k | whisper-base multilingual (eot=50257, sot=50258, transcribe=50259, no_timestamps=50261, hu=50299) | L295-301: tokenizer.json valós ID feloldása + fallback (azonos értékek); `if None in ids.values(): raise` | ✅ |
| I/O név detektálás | session input/output nevek alapján | L242-265: encoder/decoder I/O detektálás névalapú + env-felülírás (`WHISPER_QNN_ENC_IN/OUT`, `DEC_IN_T/E`, `DEC_OUT`) | ✅ (lásd 3.4 kockázat) |
| Dependency guard | onnxruntime-qnn Windows ARM64 + Hexagon driver | L85-114: egyszeri import + cache, Device Guard / DLL hiba kezelés | ✅ |

### transcript_processor.py — geniex/npu provider (geniex-npu skill)
| Ellenőrzendő | Skill tény | Kód állapota | OK? |
|---|---|---|---|
| Szerver | `geniex serve --host 0.0.0.0:18181` → `http://localhost:18181/v1` | L53: `GENIE_BASE_URL` default `http://127.0.0.1:18181/v1` | ✅ |
| Modell | `qualcomm/Qwen3-4B-Instruct-2507` (NO quant suffix, mert `:W4A16` 500-at dob) | L54: `GENIE_MODEL = "qualcomm/Qwen3-4B-Instruct-2507"` | ✅ |
| OpenAI-kompat | `/v1/chat/completions`, `model` = listából | L680, L695-700: `/chat/completions`, `model` mező | ✅ |
| #1294 (thinking a content-ben) | per-request `enable_thinking`/stb. figyelmen kívül; workaround = `<think:6124c78e>` strip a content-ből | L850/858: `no_think=True` → L147 `prefix="</think:6124c78e>\n"` a prompt elejére + L714-715: `<think:6124c78e>.*?</think:6124c78e>` regex strip a content-ből | ✅ |
| Ollama `options` | GenieX OpenAI server NEM értelmezi | L851/859: `drop_options=True` → L703 `if not drop_options` nem küldi `options` | ✅ |
| #777 (nincs tool_calls) | GenieX nem ad tool_calls-t | kód nem használ tool_calls-t (szöveges extrakció + `_text_to_summary`) | ✅ |

### npu_routes.py — ASR default
| Ellenőrzendő | Kód állapota | OK? |
|---|---|---|
| GenieX-stack Whisper-Base QNN default ASR | L80 `WHISPER_BACKEND` default `"qnn"`; L105-107 default ág `transcribe_qnn` → `whisper-base-qnn` | ✅ (de lásd 3.1 a `cpp` ág) |
| GenieX qairt NINCS ASR → külön Whisper-Base QNN | L93 docstring helyesen írja | ✅ |

**Konklúzió:** A skill-faktszintű ellenőrzés **teljesen zöld**. A GenieX és Whisper-Base QNN integráció a dokumentált tények szerint helyes.

---

## 3. Talált bug-ok / hiányosságok

### 3.1 [KÖZEPES] A `cpp` (whisper.cpp) fallback az API-n keresztül ELÉRHETETLEN + eltérő env-var név

**Hely:** `npu_routes.py` L80, L97, L100-107 vs `whisper_npu.py` L38, L386-394.

- `whisper_npu.py` két backendet támogat: `qnn` (default) és `cpp` (whisper.cpp), váltó: `WHISPER_NPU_BACKEND` (L38). A `transcribe_npu()` wrapper (L386) helyesen kezeli a `cpp`-t (`_transcribe_cpp`).
- `npu_routes.py` viszont **saját**, eltérő nevű env-változót használ: `WHISPER_BACKEND` (L80), és az `/transcribe` endpoint **csak a `"nexa"` értéket ágaztatja le** (L100). Bármi más (így `"cpp"` is) a default ágba esik, ami közvetlenül `transcribe_qnn`-t hívja (L105-106) — tehát a `cpp` backend **soha nem fut le**.
- Eredmény:
  1. `WHISPER_BACKEND="cpp"` hatástalan: QNN fut helyette (a router nem hívja `transcribe_npu`-t, ami olvasná a `WHISPER_NPU_BACKEND`-t).
  2. A két modul **két különböző env-nevet** használ ugyanarra a backend-választóra (`WHISPER_BACKEND` vs `WHISPER_NPU_BACKEND`) → nem szinkronizálható.
  3. `whisper_npu.is_npu_available()` a `WHISPER_NPU_BACKEND`-t jelenti (`backend` kulcs), miközben a router a `WHISPER_BACKEND`-t használja → a `/status` válasz és a tényleges útválasztás eltérhet.

**Javítás:** A routerben hívja meg `transcribe_npu`-t (ami olvassa `WHISPER_NPU_BACKEND`), vagy egységesítse az env-nevet (`WHISPER_NPU_BACKEND`) és ágaztassa le a `"cpp"` értéket is (`_transcribe_cpp` / `transcribe_npu(cpp)`).

### 3.2 [KÖZEPES] `omnineural` ág `NameError` (transcript_processor.py)

**Hely:** `transcript_processor.py` L889-892, `process_transcript()`.

```python
889  if provider == "omnineural" and not cfg.get("model"):   # cfg még NINCS definiálva!
890      cfg["model"] = OMNINEURAL_MODEL
891
892  cfg = PROVIDER_CFG[provider]                              # cfg itt jön létre
```

`cfg` az L889-en hivatkozva, de csak L892-ben van hozzárendelve. Bármely `provider=="omnineural"` hívás `NameError: name 'cfg' is not defined`-t dob. (A geniex/npu ág nem éri el ezt a sort, mert az `if` hamis — így a fő útvonal nem sérül, de az omnineural provider használhatatlan.)

**Javítás:** A `cfg = PROVIDER_CFG[provider]` hozzárendelést tegye az L889 *elé*, vagy a fallback-et írja át `PROVIDER_CFG[provider]["model"] = ...`-re.

### 3.3 [KISEBB] Stale dokumentáció / "GenieAPIService" elnevezés-maradványok

- `.env.example` (L7-8): `GENIE_BASE_URL=http://127.0.0.1:8912/v1` és `GENIE_MODEL=llama3.1-8b-8380-qnn2.38`. Ez a **régi** GenieAPIService (port 8912, llama3.1-8b) — a migráció után a GenieX (`:18181`, `Qwen3-4B`) a helyes. A kódalapértelmezés ugyan jó (18181/Qwen3-4B), de aki az `.env.example`-t másolja, a régi 8912-es szervert célozza meg. (CLAUDE.md is még a régi architektúrát írja le — 8912/llama3.1-8b, GenieAPIService.)
- `npu_routes.py` user-facing szövegek még "GenieAPIService"-t írnak: L148 `@npu_router.get("/genie/models", summary="GenieAPIService modelljei")`, L156 `detail=f"GenieAPIService nem érhető el..."`, L161 `summary="GenieAPIService liveness check"`. A migráció neve **GenieX**; ezek félrevezetők az operátor számára.

**Javítás:** `.env.example` és CLAUDE.md frissítése GenieX értékekre; a user-facing stringekben "GenieAPIService" → "GenieX".

### 3.4 [KISEBB / KOCKÁZAT] Decoder I/O detektálás — KV-cache feltételezés

**Hely:** `whisper_npu.py` L246-267 (`_load_qnn_sessions`).

A decoder input-nevek névalapú detektálása (`token`/`input_id` → tok_in; `encoder`/`embedding`/`hidden`/`cross` → enc_in_name) és a greedy loop (L361-369) **KV-cache nélkül** fut: minden lépésben a teljes token-sorozatot + encoder kimenetet adja át, a `past`/cache inputot nem. Ez a skill szerinti "KV-cache nélkül is gyors NPU-n" feltételre épül, és a `qualcomm/Whisper-Base` PRECOMPILED_QNN_ONNX esetén valószínűleg stimmel. **Kockázat:** ha a letöltött bundle decoder-e kötelező `past_key_values`/`past_*` inputtal rendelkezik, az `InferenceSession.run()` "missing input" hibával elszáll — az env-override (`WHISPER_QNN_DEC_IN_*`) ilyenkor csak a *nevet* írja felül, a *cache érték* etetését a kód nem implementálja.

**Javaslat:** A tényleges bundle-re ellenőrizni kell a decoder input-listát; ha van cache-input, vagy KV-cache támogatást kell írni, vagy dokumentálni, hogy csak a cache-mentes decoder bundle támogatott.

### 3.5 [INFO] `GENIE_TIMEOUT` két különböző default értéke

- `npu_routes.py` L25: `GENIE_TIMEOUT = float(os.getenv("GENIE_TIMEOUT", "10"))` (health-check, 10s OK)
- `transcript_processor.py` L55: `GENIE_TIMEOUT = float(os.getenv("GENIE_TIMEOUT", "120"))` (LLM extrakció)

Ugyanaz az env-név, két modulban két default. Működik (health vs LLM külön cél), de árnyékolás; a `.env`-ben egyszer kell beállítani és az LLM-re (120) érdemes hangolni. Csak dokumentációs/olvashatósági megjegyzés.

### 3.6 [INFO] Lokális Qwen3 hőmérséklet inkonzisztencia

`_extract_text_local` (L688): `temperature = 0.6 if family=="reasoning" else 0.2`. A geniex Qwen3 (family="qwen3") így **0.2**-n fut. A `_build_extraction_prompts` kommentje (L188: "non-thinking mode esetén temperature=0.7 (Qwen3 HuggingFace doku)") és a cloud JSON útvonal (`_call_json_api` L776: `0.7 if "qwen" in model_name.lower()`) viszont **0.7**-et javasol Qwen3-nak. A két útvonal eltérő hőmérsékleten fut ugyanarra a modellre. Nem hiba, de a geniex minősége jobb lehetne 0.7-en. Javítás: `_extract_text_local`-ban is család szerinti hőmérséklet (qwen3 → 0.7).

### 3.7 [INFO, scope-on kívül] `chat_routes.py` is migrálva, de nincs `no_think`/`<think:6124c78e>`-strip

`chat_routes.py` L78-79: `npu`/`geniex` → `GENIE_BASE_URL` + `GENIE_MODEL` (GenieX, konzisztens). A chat útvonal azonban **nem** alkalmazza a `no_think` prefixet és nem strip-eli a `<think:6124c78e>`-t (L86-89 payload). Mivel ugyanaz a GenieX Qwen3-4B szolgálja ki, a #1294 hiba a chat válaszokban is megjelenhet (`<think:6124c78e>` a content-ben). A migrációs scope csak a 3 fájlra szólt, de érdemes a chat útvonalon is bevezetni a `<think:6124c78e>`-stripet (legalább a strip, a prefix opcionális).

---

## 4. Pozitívumok

- A GenieX provider és a Whisper-Base QNN backend **skill-faktszinten helyes** (QNN htp, forced-token sorrend, modell név quant-suffix nélkül, no_think+strip a #1294-re, drop_options a GenieX OpenAI server miatt).
- Jó védekezés: `_stderr_suppressor` (Device Guard/venv-shadowing), egyszeri onnxruntime import cache, `is_npu_available()` részletes státusz.
- A `whisper.cpp` és a legacy Nexa Parakeet fallback-ök megtartva (kód szinten), csak az API-rétegben a `cpp` ág nem éri el őket (lásd 3.1).
- Hiba-kezelés az `/transcribe`-ban rétegezett (ConnectError/HTTPStatusError/RuntimeError/ValueError/Exception → megfelelő HTTP kód).

---

## 5. Javasolt javítási sorrend

1. **3.1** — `cpp` fallback elérhetetlenség + env-var egységesítés (közepes, funkcionális).
2. **3.2** — `omnineural` `NameError` (közepes, egy sor).
3. **3.3** — `.env.example` + CLAUDE.md + user-facing "GenieAPIService" → "GenieX" (kisebb, de félrevezető).
4. **3.4** — decoder bundle I/O ellenőrzése KV-cache szempontból (kockázat, validálás).
5. **3.6 / 3.7** — Qwen3 hőmérséklet 0.7 + `<think:6124c78e>`-strip a chat útvonalon is (minőség).

---

## 6. Fájlok (vizsgált)

- `backend/app/whisper_npu.py` (425 sor) — Whisper-Base QNN + whisper.cpp backend
- `backend/app/transcript_processor.py` (1027 sor) — `npu`/`geniex` provider cfg L846-860, `_extract_text_local` L676-718, `_get_model_family` L106-126, `_build_extraction_prompts` L129-257, `process_transcript` PROVIDER_CFG L829-884 (hiba L889), health L973-981
- `backend/app/npu_routes.py` (394 sor) — `/transcribe` L83-143 (hiba L80/97/105), `/genie/*` L146-173, legacy nexa L347-394
- Melléklet: `backend/app/chat_routes.py` L60-89 (scope-on kívül, 3.7), `.env.example` L7-8 (3.3)

**Nem módosítottam a kódot** — kizárólag olvasás + ez a jelentés.
