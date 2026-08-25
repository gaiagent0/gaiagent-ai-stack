# CT-304 Bot-háló — Krónika

> A CT-304 (Debian 12 LXC, Proxmox, `10.10.40.210`) Hermes Agent konténerben futó 8 botos hálózat dokumentációja.

A dokumentációt az **Író** (iro) bot állította össze a migració során gyűjtött runbookok, handoff jegyzetek, fejlesztési terv és a `hermes-bots-herdr.sh` szkript alapján.

---

## 1. Áttekintés

A CT-304 bot-háló **8 féle szakszerűen specializált botból** áll, kettes workspace-struktúrába szervezve. A botok **képesség-alapú modellezéssel** kapott modelleket — mindegyik bot a feladatköréhez leginkább illő modellkel dolgozik.

**Cél repo:** `gaiagent-ai-stack` (account: `gaiagent0`)

---

## 2. A 8 bot — feladatköri táblázat

| # | Bot (profil neve) | Megnevezés | Fő feladatkört | Workspace | Küldetés rövidítve |
|---|-------------------|------------|----------------|-----------|---------------------|
| 1 | `rendszergazda` | Rendszergazda | Infrastruktúra, LXC/Proxmox kezelés, rendszerüzemeltetés | **w1 (ops)** | A CT-304 alatt fogyó rendszergazdálkodási feladatok |
| 2 | `orszem` | Ország- / szempillantan | Átfogó állapotfigyelés, összefoglaló betekintések | **w1 (ops)** | A háló egészének "szeme" — állapotorendszer |
| 3 | `biztonsagor` | Biztonsági őr | Biztonsági monitorozás, log-analízis, kockázatfelmérés | **w1 (ops)** | Rendszer- és adatorvényesítés, anomáliafigyelés |
| 4 | `kutato` | Kutató | AI/ML kutatás, arxiv, reddit, github figyelő | **w1 (ops)** | Ismételt és mély kutatás módszeres kidolgozása |
| 5 | `iro` | Író | Tartalomírás: cikkek, dokumentációk, összefoglalók | **w3 (tartalom)** | Olvasható, emberi 느낌ű magyar szöveg előállítás |
| 6 | `hirado` | Híradó | Tech-hírek szerkesztése, Telegram broadcast | **w3 (tartalom)** | Friss AI/tβch hírek gyors városólása és cikk-szállítás |
| 7 | `fejleszto` | Fejlesztő | Python/backend speciális fejlesztés, architektúra | **w3 (tartalom)** | Kód-architektúra tervezés és komplex backend feladatok |
| 8 | `kodolo` | Kódoló | Kódgenerálás, refactoring, agentes kód-feladatok | **w3 (tartalom)** | Fókuszált, lokalizálható szoftverkód-előállítás |

---

## 3. Modellek képesség-alapú elosztása

A hálózat **5 modellegyüttessel** (6 variánssal) működik. Az elosztás szempontjai:

- **Kontextusablak mérete** (nagy kutatási/security feladatok)
- **Kód- és agentikum-képesség** (fejlesztő/kódoló)
- **Continuum gyorsaság** (rendszergazda CLI/felületi műveletei)
- **Általános és nyelvspecifikus írás** (tartalom workspace)
- **Multimodális befogadás** (kutatás, biztonság)

### 3.1 Modell-profilok

| Modell | Képességek (lényeges) | Kontextus | Paraméterek / családhát | Ajánlott gebruik |
|--------|-----------------|----------|----------------------------|------------|
| **ox-alpha** (Stealth) | 1·048·576 tok kontextus, multimodal (txt+img+video), erős reasoning (MMLU ~97 jel), tool-calling, kódszers지기, agentes munka. 2026-08-20 release. | 1M tok | Névtelen fejlesztő; openrouter / stealth API (jelenleg ingyenes preview) | **Kutató** (nagy kutatási anyagok, multimodal papírok); **Biztonsági őr** (masszív log/security adat eleme, 1M kontextusban) |
| **LongCat-2.0** (Meituan) | Trilliós paraméter-tér,MoE, sparse attention, agentic coding 특화, repository-méretű módosítás, codebase migration, tool-heavy feladatok. | N/A (nagy) | Meituan LongCat család | **Fejlesztő** (repo-nagyitas, web app fejlesztés, agentes kód-alapú feladatok) |
| **solar-pro4** (Upstage AI) | Általános célokra, magyar nyelv támogatás, megbízható generáció, szövegalapú feladatok. | Ált. | Upstage AI, in-house | **Ország- / szempillantan** (figyelés, összefoglalás); **Író** (magyar tartalomírás); **Híradó** (hírösszesítés, broadcast előkészítés) |
| **Laguna XS 2.1** (Poolside) | 33B teljes / 3B aktivált MoE, agentic coding 특례, SWE-bench Multilingual +5.4%, terminal-task erős, interleaved thinking, helyi futtatás (36 GB RAM Mac-en is). Apache-2.0 súl. 256K kontextus. | 262·144 tok | Poolside, open-weights (Apache-2.0) | **Kódoló** (fókuszált kód-generálás, refactoring, agentic coding — kompakt, lokalizálható) |
| **Laguna S 2.1** (Poolside) | 118B teljes / 8B aktivált MoE, nagyobb táradat, 70.2% Terminal-Bench 2.1, ugyanaz a laguna architektúra mint XS, 256K kontextus. Nem placebo — de szükség van min. 75 GB RAM/helyettesítő GPU. OpenMDW-1.1. | 256·144 tok | Poolside, open-weights (OpenMDW-1.1) | **Fejlesztő alternatíva / uplift** (ha a LongCat helyett+nal nagyobb coding tárkapacitas kell, vagy a kodolo nehezebb feladataira) |
| **Step-3.5/3.7 Flash** (StepFun) | 196B teljes / 11B aktivált MoE, gyors inference, erős terminal/ops teljesítés (SWE-bench Verified ~60-74%, LiveCodeBench ~86), 256K kontextus, 3-way MTP 지음 gyorsítás. | 256·144 tok | StepFun (stepfun-ai), OpenRouter ($0.1-0.2 / 1M input) | **Rendszergazda** (CLI automatizálás, Proxmox/LXC parancsok, gyors válaszidő, tool-use infrastruktúra feladatok) |

### 3.2 Elosztási táblázat

| Bot | Kitählt modell | Miért ez? — képesség-alapú indoklás |
|-----|----------------|--------------------------------------|
| `rendszergazda` | **step-flash** | Gyors inference, erős terminal/ops benchmark (LiveCodeBench 86.4), 3-way MTP gyorsasága, tool-calling infrastruktúra parancsokhoz. LXC/Proxmox management, gyors CLI válaszidő. |
| `orszem` | **solar-pro4** | Általános irányítási és figyelési feladatokhoz megbízható, magyar összefoglalók, állapotjelzők értelmezése. In-house modell, stabil. |
| `biztonsagor` | **ox-alpha** | 1M tok kontextus a nagy log-fájlok / security adathalmok elezéséhez; multimodal (screenshot / diagram elemzés); erős reasoning a kockázat-felismerésben. |
| `kutato` | **ox-alpha** | 1M kontextus a hosszú kutatási anyagok / teljes papírok feldolgozásához; multimodal kutatási diagramok, ábrák befogadásához; módszeres, hosszabb agentes kutatási munkafolyamatok. |
| `iro` | **solar-pro4** | Magyar nyelvű tartalomírás, cikkek, dokumentációk, összefoglalók. Stabil, általános szövegalkotó, emberi hangvételű előállítás. |
| `hirado` | **solar-pro4** | Hírösszesítés, gyors szerkesztés, broadcast előkészítés. Általános és gyors, magyar tartalomszerkesztési feladatokhoz optimális. |
| `fejleszto` | **longcat** (fő); **laguna-s** (alternatíva/uplift) | LongCat-2.0: trilliós MoE, repository-méretű kód-aldaú, agentic coding, codebase migration, web app fejlesztés. Ha nagyobb coding tárkapacitas kell: Laguna S 2.1 (118B/8B, 70.2% Terminal-Bench). |
| `kodolo` | **laguna-xs** | 33B/3B aktivált MoE, célzott agentic coding, SWE-bench Multilingual +5.4%, helyi futtatás (36 GB RAM), kompakt és specializált kódgenerálás. Interleaved thinking + tool-calling a kód-feladatokhoz. |

> **Megjegyzés a modell-kínálathoz:** A `laguna-s/xs` notation a Poolside Laguna család két többszörösét jelöli — XS a **kodolo**é (kompakt, lokalizálható), S az **fejleszto** mögötti uplift lehetőség. Jelenleg a fejlesztő fő modellje a **longcat**, a Laguna S fallback/upgrade opció.

---

## 4. herdr workspace struktúra

A 8 bot **két workspace-ben** renderszve a `hermes-bots-herdr.sh` szkript révén.

### 4.1 Workspace W1 — „ops"

| Pane | Bot (profil) | Szerepe a W1-ben |
|------|--------------|------------------|
| `w1:p1` (root) | **rendszergazda** | Nézet leadása, infrastruktúra irányítás |
| `w1:p2` (jobbra) | **orszem** | Állapotfigyelés, visszatekintő betekintés |
| `w1:p3` (le) | **kutato** | Kutatási folyamatok, material gyűjtés |
| `w1:p4` (jobbra) | **biztonsagor** | Biztonsági monitor, anomália-vizsgálat |

> SSH-ről: `ssh root@10.10.40.210` majd `herdr` — a W1 a **első** workspace (Ctrl+B + 1).

### 4.2 Workspace W3 — „tartalom"

| Pane | Bot (profil) | Szerepe a W3-ban |
|------|--------------|------------------|
| `w3:p1` (root) | **iro** | Tartalomgyártás alapja, cikk/szerkesztés |
| `w3:p2` (jobbra) | **fejleszto** | Fejlesztési támogatás, architektúra |
| `w3:p3` (le) | **hirado** | Híradás, szerkesztett híreket szolgáltató |
| `w3:p4` (jobbra) | **kodolo` | Kód-alad upper output, gyors prototípusok |

> W3 a **műszerfal második workspace** (Ctrl+B + 2 a W1 után, vagy a sidebar kattintás).

### 4.3 Váltás és összeműködés

- **W1 ↔ W3 váltás:** `Ctrl+B` + `1` (W1) vagy `2` (W3), illetve sidebar kattintás
- **Sidebar kibontakoztatása:** `Ctrl+B` + `b`
- **Pane váltás:** egérkattintás vagy `Ctrl+B` + nyílbillentyű
- A két workspace közötti **bot-to-bot kommunikáció** tervezett (lásd FEJLESZTESI_TERV.md), pl. kutató → híradó pipeline `context_from` cron paraméterrel.

---

## 5. Fájlstruktúra és további dokumentáció

A CT-304 beolvasási építészet az alábbi forrásfájlokból áll össze:

| Fájl / hely | Tartalom |
|-------------|----------|
| `/root/.hermes/CT304_MIGRATE_RUNBOOK.md` | Migrációs runbook — lépések, parancsok, ellenőrzések |
| `/root/.hermes/HANDOFF.md` | Handoff jegyzetek — átadás, állapot, nyitott kérdések |
| `/root/.hermes/CT304_TO_LAPTOP_ACCESS.md` | Laptop SSH hozzáférés beállítása |
| `/root/hermes-bots/FEJLESZTESI_TERV.md` | Bot-háló fejlesztési terv — profilok, skillek, bot-to-bot messaging, MCP, priority lista |
| `/root/bin/hermes-bots-herdr.sh` | A 8 botos herdr launcher szkript (W1 + W3 építése) |
| `/root/hermes-bots/biztonsagor/` | A biztonsági őr profil tállapja (clawsentry_monitor.sh, SOUL.md) |

---

## 6. Fejlesztési irányok (tovább)

A `FEJLESZTESI_TERV.md` alapján a következő akciók parkolóban:

1. **Valódi bot profilok létrehozása** — `hermes profile create <bot>` minden 8 botra
2. **Per-bot model-pin** — a fenti táblázat szerinti modellezés profil-levelen
3. **Per-bot skillek** — pl. kutató ↔ Reddit scraping workflow (`/learn`)
4. **Bot-to-bot messaging** — pl. `@kutato` címkézet a híradón, context_from cron pipeline
5. **MCP szerverek per-bot** — pl. obsidian-vault (kutató), proxmox API (rendszergazda), github MCP (fejleszto)
6. **Group chat** — „Heti briefing" (kutató + híradó + írő), „Infra" group (őrző + gazda + biztos)
7. **Profile distributions** — `hermes profile export` → git repo → újratelepíthető

---

*Legutóbb frissítve: 2026-08-25 (migráció után)*
