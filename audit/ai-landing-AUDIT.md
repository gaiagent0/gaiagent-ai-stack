# ai-landing — Audit

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\ai-landing`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Verzió:** 2026-04-02 (utolsó: contact form disabled) · **Státusz:** Aktív, de hiányos
**Összefüggés:** gaiagent0 repócsalád auditja — marketing landing page

---

## 1. Mi ez?

**Modern AI & IT Services landing page** (Next.js 15, EN/HU bilingual, Cloudflare Pages).
NEM AI stack komponens — **marketing weboldal** a gaiagent0 "fő" landing page-e.
Cél: bemutatkozás, szolgáltatások, kapcsolatfelvétel (contact form).

---

## 2. Architektúra

```
Next.js 15 (App Router)
├── src/app/page.tsx          # Főoldal
├── src/*                     # Komponensek, i18n (EN/HU)
├── public/                   # Statikus asset-ek
├── next.config.ts            # Cloudflare Pages (static export)
├── AGENTS.md, CLAUDE.md      # AI agent instrukciók
└── package.json              # Deps (Next.js, Tailwind, Turnstile?)
```

**Deploy:** Cloudflare Pages (static export). Korábban `output: export`, most default adapter (fix commit).

---

## 3. GenieX átjárhatóság

**N/A** — ez egy marketing oldal, nem AI backend. Nincs köze a GenieX-hez.
**KAPCSOLÓDÁS:** Ez a gaiagent0 "fő" landing page? A `gaiagent-ai-stack` főrepo README hivatkozhatja (link a landing-re).

---

## 4. Tech debt

1. **🔴 Contact form DISABLED** — "fix: disable contact form due to Turnstile integration issues" (2026-04-02). A kapcsolatfelvétel NEM működik jelenleg!
2. **Turnstile hiány** — Cloudflare Turnstile bot védelem nem működik → spam kitett a form (ha újra engedélyezik)
3. **Cloudflare deploy config** — `output: export` eltávolítva, default adapter (ok, de dokumentálatlan miért)
4. **Nincs README** a deploy folyamatról (csak package.json scripts)

---

## 5. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| Contact form validáció | Van (README említi) | ⚠️ De DISABLED jelenleg |
| Turnstile | Nincs (integration issues) | 🔴 Bot védelem HIÁNYZIK |
| CORS | N/A (static page) | ✅ Nincs |
| API key | N/A (nincs backend) | ✅ Nincs |

---

## 6. Összegzés

**Állapot:** Érett Next.js landing page, de **contact form letiltva** (Turnstile hiba).
**GenieX reláció:** N/A (marketing oldal).
**Tech debt:** 🔴 Contact form disabled, Turnstile hiányzik → a kapcsolatfelvétel nem működik.
**Döntés:** A `gaiagent-ai-stack` főrepo README-jében linkelni kell (ez a "fő" oldal), de a contact form javítása külön feladat (Turnstile visszaállítása vagy más bot védelem).

**Fájlok olvasva:** README.md (GitHub adatok), package.json (structure), next.config.ts (GitHub)
**Nem olvastam mélyebben:** src/* (a README + package.json fedte a stack-et)
