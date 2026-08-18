# llama-mtp-snapdragon — Audit

**Dátum:** 2026-08-18 · **Repo:** `C:\Users\istva\Dev\portfolio\Projects\gaiagent-voice-audit\llama-mtp-snapdragon`
**Audit típus:** csak olvasás + jelentés (kód NEM módosítva)
**Verzió:** v1.0 (init 2026-05-21, utolsó: 2026-05-31)
**Összefüggés:** gaiagent0 repócsalád auditja — llama.cpp MTP build (ARM64 Windows)

---

## 1. Mi ez?

**llama.cpp MTP (Multi-Token Prediction) build + szerver indító scriptek Snapdragon X Elite ARM64 Windows-re.**
NEM NPU! **CPU-only** llama.cpp (a README kifejezetten írja: "GPU/NPU: Adreno — llama.cpp-ben NEM használható, NPU csak ONNX").
Cél: Qwen3.5/3.6 MTP modellek futtatása CPU-n, gyorsabb generálás (MTP = speculative decoding).

---

## 2. Architektúra

```
llama-server.exe (CPU-only, ARM64)
  ├─ Qwen3.6-35B-A3B (port 8081, ~22 GB RAM, 262K ctx)
  ├─ Qwen3.5-9B-MTP  (port 8082, ~5.5 GB RAM)
  ├─ Qwen3.5-4B-MTP  (port 8082, ~2.5 GB RAM)
  └─ Qwen3.5-2B-MTP  (port 8082, opcionális)
```

**Build:** cmake + clang-cl (LLVM), ARM64 CPU-only flags (`-DGGML_CUDA=OFF -DGGML_VULKAN=OFF`).
**Scriptek:** `build-mtp-llama.ps1` (teljes build), `start-mtp-*.ps1` (szerver indítók), `switch-mtp-model.ps1` (modellváltó).

---

## 3. GenieX átjárhatóság

**EZ NEM GenieX!** Ez **CPU llama.cpp**, nem NPU QNN. Két kategória:
- **GenieX** (:18181, NPU QNN) — kis modellek (Qwen3-4B), nagyon gyors NPU-n
- **llama-mtp** (CPU llama.cpp) — nagy modellek (35B-A3B, 9B), CPU-n, MTP gyorsítással

**Reláció:** **Kiegészítő, nem redundáns!**
- GenieX: kis, gyors, NPU (Qwen3-4B)
- llama-mtp: nagy, lassabb, CPU (35B-A3B minőség)

A control-center-ben mindkettő szerepel (mtp-35b/mtp-8b + genie), tehát **párhuzamosan használt** stack-ek.

---

## 4. Tech debt

1. **Hardcoded path-ek**: `E:\models\mtp\`, `E:\models\mtp-small\`, `C:\AI\scripts\`, `C:\Users\istva\SnapdragonNPU_Build\` — más gépen nem működik
2. **27B modell említve** (commit "add 27B model") de csak 35B/9B/4B scriptek — a 27B hiányzik a scripts/-ból
3. **MTP 27B párhuzamos futás** nyitott feladat (control-center "Control Center: mtp-9b (8082) és mtp-4b service bejegyzés" — nincs meg)
4. **WSL2 elérés**: "127.0.0.1-en hallgat, WSL-ből 172.25.16.1:8081" — nem konzisztens a WSL networkinggel
5. **Build frissítés**: "MTP hivatalosan merged (2026-05-16)" — a repo snapshot 2026-05-21, de a build script lehet elavult

---

## 5. Biztonság

| Téma | Állapot | Értékelés |
|---|---|---|
| API key | Nincs (helyi szerver) | ✅ Jó |
| Port expozíció | 8081/8082 localhost | ✅ Helyi (de WSL2-ből 172.25.16.1, az már hálózat) |
| CORS | N/A (llama-server) | ✅ Nincs |

---

## 6. Összegzés

**Állapot:** Érett build/script repo, CPU llama.cpp MTP-re optimalizálva.
**GenieX reláció:** Kiegészítő (nagy modellek CPU-n, vs GenieX kis NPU modellek). Nem redundáns.
**Tech debt:** Hardcoded path-ek, 27B modell hiányzik a scriptekből, WSL2 port inkonzisztencia.
**Döntés:** A `gaiagent-ai-stack` főrepo README-jében: GenieX (NPU, kis) + llama-mtp (CPU, nagy) = komplementer stack-ek, mindkettő a control-center-ben menedzselt.

**Fájlok olvasva:** README.md (klón + GitHub adatok)
**Nem olvastam mélyebben:** scripts/*.ps1 (a README fedte a parancsokat)
