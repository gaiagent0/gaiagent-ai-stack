# hermes-foundry-npu — Audit

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\hermes-foundry-npu`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Verzió:** v1.0 (init 2026-05-31, utolsó: 2026-06-01)
**Összefüggés:** gaiagent0 repócsalád auditja — Hermes + Foundry NPU integráció

---

## 1. Mi ez?

**Hermes Agent + Foundry Local NPU integrációs dokumentáció és konfig repo.**
NEM futtatható alkalmazás — docs + config + skill + profil + script gyűjtemény.
Cél: Hermes Agent (Windows) NPU-gyorsított LLM használata Foundry Local-on (Snapdragon X Elite, vivo2 gép).

---

## 2. Architektúra (3 réteg)

```
Hermes Agent (Windows)
  └─ npu-agent profil → custom provider: foundry-npu
       └─ base_url: http://localhost:4001/v1 (LiteLLM proxy)
            └─ LiteLLM Proxy (:4001) → openai/qwen2.5-7b-instruct-qnn-npu:2
                 └─ api_base: http://localhost:5272/v1
                      └─ Foundry Local Service (:5272) → QNN EP (Hexagon HTP)
                           └─ NPU: Qwen2.5-7B (tool calling) / DeepSeek-R1-7B (reasoning)
```

**Konzisztens?** Igen — a stack leírja a teljes utat Hermes → NPU-ig. A LiteLLM proxy (:4001) logikus központ.

---

## 3. GenieX átjárhatóság

**EZ Foundry Local (nem GenieX)!** Két külön NPU stack létezik a gaiagent0 univerzumban:
- **Foundry Local** (:5272, QNN EP) — ez a repo (Qwen2.5-7B, DeepSeek-R1-7B)
- **GenieX** (:18181, Qualcomm AI Engine Direct) — a meetcore új célja (Qwen3-4B)

**Reláció:**
- A meetcore a **GenieX Qwen3-4B**-re váltott (újabb, :18181)
- Ez a repo a **Foundry Local Qwen2.5-7B**-t használja (régebbi, :5272)
- **MINDKÉT helyi NPU stack**, de más technológia. Nem redundáns, hanem **alternatív útvonalak**:
  - GenieX = Qualcomm natív AI Engine Direct (újabb, Qwen3-4B)
  - Foundry Local = Microsoft QNN EP wrapper (Qwen2.5-7B, DeepSeek-R1-7B)

**Döntés:** A meetcore (GenieX) a referencia most. Ez a repo (Foundry) **másik opció** — ha a GenieX nem elérhető, a Foundry Local is NPU-t ad. A kettő párhuzamosan is futhat.

---

## 4. Tech debt

1. **Legacy GenieAPIService :8912 említés** (README "Kapcsolodo stack") — a GenieAPIService a legrégebbi stack (llama3.1-8b), már nem a cél
2. **Foundry vs GenieX keveredés** — a repo nem említi a GenieX-et, holott a meetcore azt használja. Dokumentációs hiány: melyik a "fő" NPU stack?
3. **Hardcoded path-ek**: `E:\models\foundry\`, `C:\AI\apps\Foundry\`, `C:\Users\istva\...` — más gépen nem működik
4. **ChromaDB**: `C:\Users\istva\Documents\gaiagent\chromadb_data\` — Obsidian RAG integráció említve, de nincs benne a repóban

---

## 5. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| API key (.env) | `LITELLM_MASTER_KEY` helyi | ✅ Lokális, .env-ben |
| API key kezelés | `.gitignore` van? | ⚠️ Ellenőrizendő (check_secrets.py ignore-olva a .gitignore-ban) |
| Port expozíció | 4001/5272 localhost | ✅ Helyi |

---

## 6. Összegzés

**Állapot:** Érett dokumentációs repo, konzisztens Foundry Local stack-leírással.
**GenieX reláció:** Foundry Local = alternatív NPU stack (nem GenieX). Mindkettő helyi NPU, párhuzamosan használható.
**Tech debt:** Legacy GenieAPIService említés, Foundry vs GenieX dokumentációs tisztázás szükséges.
**Döntés:** A `gaiagent-ai-stack` főrepo README-jében tisztázni kell: **GenieX = elsődleges NPU (meetcore), Foundry Local = másodlagos (ez a repo)**.

**Fájlok olvasva:** README.md, CHANGELOG.md (GitHub adatokból), structure (klón)
**Nem olvastam mélyebben:** skill/SKILL.md, profiles/npu-agent/* (a README fedte a stack-et)
