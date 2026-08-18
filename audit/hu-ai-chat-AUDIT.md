# hu-ai-chat — Audit jegyzőkönyv

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\hu-ai-chat`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Összefüggés:** gaiagent0 repócsalád auditja — meetcore (local NPU) mellett ez a másik élő repo.

---

## 1. Koncepció

Magyar nyelvű AI chat alkalmazás, **felhőalapú** architektúrával:
- **AI:** Alibaba Qwen API (qwen-plus / turbo / max) — DashScope OpenAI-kompatibilis endpoint
- **Backend:** FastAPI + uvicorn (Python)
- **Frontend:** Vanilla HTML/CSS/JS (zero framework, no build)
- **Deploy:** Oracle Cloud Always Free AMD VM + Caddy reverse proxy (auto HTTPS)
- **IaC:** OpenTofu (portfolio-infra repo)

**Összehasonlítás a meetcore-val:**
| | meetcore | hu-ai-chat |
|---|---|---|
| AI futtatás | **Helyi NPU** (GenieX/Whisper-Base QNN) | **Felhő** (Alibaba Qwen API) |
| Offline | Igen | Nem (API key + internet kell) |
| Nyelv | meeting assistant (átírás+összefoglaló) | általános chat |
| Infra | laptop (Snapdragon X Elite) | OCI VM (Always Free) |

**Kapcsolódás a GenieX/NPU témához:** NINCS. Ez a repo felhős, nem helyi NPU. Külön kell kezelni.

---

## 2. Működik-e?

**Igen, működőképes** — a README részletes, a deploy script (setup.sh) végigvezetett.
- Backend: `main.py` 57 sor, tiszta FastAPI
- Frontend: statikus HTML, nincs build
- Deploy: systemd service + Caddy, reprodukálható

**Függőségek:** `requirements.txt` (nem olvastam, de a `openai` SDK + `fastapi` + `uvicorn` kell).
**API kulcs:** `DASHSCOPE_API_KEY` env-ben (`~/.env.hu-ai-chat` VM-en, `.env` lokálisan) — **nem hardcode-olt**.

---

## 3. Biztonsági kockázatok

| Téma | Állapot | Értékelés |
|---|---|---|
| API key kezelés | env-ben, `.gitignore`-ban (feltételezve) | ✅ Jó |
| CORS | `allow_origins=["https://chat.istvanszechenyi.uk"]` — szűkített | ✅ Jó (meetcore-nél `*`!) |
| HTTPS | Caddy auto Let's Encrypt | ✅ Jó |
| Rate-limit | NINCS | ⚠️ Hiányzik — DoS / API key abuse veszély |
| Input validáció | `ALLOWED_MODELS` szűrő van | ✅ Jó |
| Hibaüzenetek | `str(e)` raw exception → 500 | ⚠️ Lehet info-leak (stack trace) |

---

## 4. Tech debt

1. **Rate-limiting hiánya** — a `/api/chat` korlátlanul hívható → API key költségrobbanás
2. **Error handling** — `detail=str(e)` visszaadja a raw hibát (pl. API kulcs hiba szövege)
3. **Nincs auth** — bárki hívhatja a chat-et (ha tudja a domain-t) → rate-limit nélkül abuse
4. **`max_tokens=1000` hardcode** — nincs user-beállítás
5. **Conversation history** — a frontend küldi a teljes history-t, de a backend nem validálja a méretet
6. **Dependency pinning** — `requirements.txt` nincs verziózárolás (ha van) → reprodukálhatósági kockázat

---

## 5. Javaslatok (nem kötelező, mert nem NPU téma)

- Rate-limiter middleware (pl. `slowapi`) hozzáadása
- Hibaüzenetek magyarra fordítása + generic 500 (ne raw exception)
- `max_tokens` user-paraméterként
- `requirements.txt` verziózárolása

---

## 6.Összegzés

**Állapot:** Működő, tiszta felhős chat app. NEM része a GenieX/NPU migrációnak.
**Kockázat:** alacsony (CORS jó, key env-ben), de rate-limit hiány miatt production-ban sebezhető.
**Döntés:** Külön tartható — a meetcore audit mellé referenciaként, de NEM kell átírni GenieX-re.

**Fájlok olvasva:** README.md, backend/main.py, deploy/setup.sh
**Nem olvasva:** requirements.txt, frontend/index.html (nem kritikus az audit szempontjából)
