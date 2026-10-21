# PRAMAAN — Digital Evidence Integrity & Authenticity Platform (PS-05)

**From file hash to court-ready forensic opinion, fully offline.**

PRAMAAN receives digital evidence (photos, CCTV/video, audio). It proves the file is **unchanged** since intake (integrity) and tests whether the content is **genuine** (authenticity) with many independent, explainable forensic signals. The output is a digitally signed, bilingual (English + हिन्दी) report that pre-fills the particulars for the expert certificate under **Section 63, Bharatiya Sakshya Adhiniyam 2023**.

```
Upload / demo kit ─► SHA-256/SHA-1/MD5 + write-once store ─► hash-chained custody ledger
   ─► modality router ─► image │ CCTV/video │ audio │ metadata │ cross-modal (lips vs speech)
   ─► evidence fusion (families, calibrated LLRs, abstention)
   ─► Next.js dashboard (UV-lamp compare, timelines, EN/हिन्दी) ─► signed PDF + QR verification
```

## What makes it different (USPs)

| # | USP | Why judges care | Where in code |
|---|-----|-----------------|---------------|
| 1 | **Integrity ≠ authenticity.** Hash-chained, append-only custody ledger with signed Merkle anchors; analysis only on copies; original re-hashed after every job. | Maps onto the Section 63 hash requirement; any edited ledger row is detected. | `custody/`, `pipeline.py` |
| 2 | **CCTV on-screen-clock reconciliation.** OCR of the burned-in DVR clock against container timestamps, keyframe cadence, the re-encoding footprint (VPF) and frame/noise evidence. Catches deletion even when re-timed, reordering, loop inserts and freezes. | CCTV tampering is temporal; most teams only run face-deepfake models. | `analyzers/video/` |
| 3 | **India-specific ENF.** Phase continuity of the 50 Hz mains hum (all harmonics) locates audio cuts; a 60 Hz hum flags audio not recorded on the Indian grid. The test abstains when the hum is too weak or noisy. | Blind, explainable, no AI model needed. | `analyzers/audio/enf.py`, `scripts/enf_logger.py` |
| 4 | **Cross-modal lip-sync.** Mouth motion vs speech energy: a constant offset means the audio was re-laid; a mid-clip break means words were replaced. | Catches doctored statements that pass audio-only and video-only checks. | `analyzers/video/lipsync.py` |
| 5 | **Explainable fusion with abstention.** Each test gives an LLR and a reliability for *this* input; output is SWGDE-style categories, never "87 % fake". | Court-defensible; false accusations are worse than abstaining. | `fusion.py` |
| 6 | **Signed, bilingual reports.** Ed25519-signed manifest, QR verification page, offline verifier, Hindi summary rendered with proper Devanagari shaping, Section 63 Part-B draft annex. | Report tampering is detectable; Hindi-speaking IOs and courts can read it. | `report/` |
| 7 | **Judge-ready demo kit.** 32 one-click test exhibits: 16 real-world files from public research datasets and 16 generated cases with known answers, pre-analysed on first start. | Judges see results in seconds without uploading anything. | `samples.py`, `components/Home.tsx` |
| 8 | **Air-gapped.** No cloud APIs or CDNs; fonts self-hosted; the optional local LLM may only rephrase. | Police data never leaves the lab. | everywhere |

## Quick start: one command

Install the four system tools first: **Python 3.10+, ffmpeg, tesseract-ocr, git**. Add **Miniconda** for TruFor and **Node.js 18+** for the Next.js build. The script checks each one and prints the exact install command for your OS if something is missing (e.g. `winget install Gyan.FFmpeg`). Then, from the project folder:

```bash
python scripts/setup_all.py --all --cpu-torch     # everything; drop --cpu-torch if you have an NVIDIA GPU
uvicorn pramaan.api.server:app --port 8000        # open http://localhost:8000
```

`--all` does the following, skipping anything already done:

| Step | What it installs | Size |
|---|---|---|
| Core | `requirements.txt` | ~150 MB |
| Deep-learning extras | PyTorch + transformers (+ c2pa, open_clip if they install) | ~700 MB CPU / ~2.5 GB GPU |
| AI-image classifier | Ateeqq/ai-vs-human-image-detector → `models/hf` | ~370 MB |
| TruFor | clone → conda env from TruFor's yaml → weights zip, **MD5-verified** → `pretrained_models/trufor.pth.tar` | ~2 GB |
| Demo samples | generated set (`demo_samples/`); real-world set is already bundled | ~80 MB |
| Next.js | `npm install && npm run build` → `frontend/out` | ~400 MB |

Other modes: `--check` (report only), `--trufor` (add TruFor later), `--trufor-zip FILE` (use a weights zip you downloaded in a browser), `--gpu` (TruFor on GPU), `--skip-ml`, `--dry-run`. Decisions such as which Python runs TruFor are saved to `models/local_settings.json`, so no environment variables are needed. The summary table at the end shows ✓ / ✗ / – for every step and every detector.

### Turn on AI-image detection (recommended)
Without it, photos from WhatsApp/Instagram and AI images usually come back **Inconclusive**, and the dashboard tells you why. Messaging apps strip camera metadata, and an AI image passes pixel and compression tests, so nothing else can give it away.
```bash
pip install -r requirements-ml.txt          # PyTorch + transformers
python scripts/install_models.py            # downloads the classifier once (~370 MB) into models/hf
export HF_HUB_OFFLINE=1                     # optional: run fully offline from then on
```
Default model: [Ateeqq/ai-vs-human-image-detector](https://huggingface.co/Ateeqq/ai-vs-human-image-detector) (SigLIP 2, Apache-2.0, 120k training images incl. Midjourney v6.1, Flux 1.1, SD 3.5, GPT-4o). Change it with `PRAMAAN_AIGI_MODEL`. Its card reports 99 % on its own test split, but real-world accuracy is lower (users report overfitting). PRAMAAN therefore weights it so a ≥97 % "AI" score gives *Indications*, never *Strong indications* on its own, and a clear "camera photo" score lets a clean image reach *No indications*.

### Install TruFor (optional, best image-edit localiser)
```bash
python scripts/setup_all.py --trufor          # add --gpu if you have an NVIDIA GPU
```
This clones [grip-unina/TruFor](https://github.com/grip-unina/TruFor), downloads its weights (MD5-checked against the published `7bee48f3476c75616c3c5721ab256ff8`), and installs the three small packages TruFor's inference imports besides PyTorch (`timm`, `yacs`, `tqdm`). It then runs TruFor once on a demo photo to prove it works.

It runs TruFor with **your current Python and its modern PyTorch**. TruFor's original 2022 conda environment (Python 3.7, PyTorch 1.11, ~2 GB) cannot load on recent Linux: glibc ≥ 2.41 refuses old PyTorch with *"cannot enable executable stack"*. It is still available via `--trufor-conda` for older systems. PyTorch ≥ 2.6 would also reject TruFor's 2022 checkpoint under its default `weights_only` rule, so PRAMAAN lifts that rule for the TruFor subprocess only, after the MD5 check. Expect about a minute per photo on CPU. TruFor's licence permits informational and non-profit use only.

### The Next.js dashboard (`frontend/`)
```bash
cd frontend
npm install
npm run build        # static export → frontend/out, served automatically by the Python server
# development with hot reload:
NEXT_PUBLIC_API=http://localhost:8000 npm run dev      # http://localhost:3000
```
The server picks the UI in this order: `frontend/out` (Next.js build), then `pramaan/web_dist` (a prebuilt bundle of the *same* React components made with `node frontend/scripts/build-harness.mjs`, so the dashboard works even before `npm install`), then the legacy single-file page.

Docker: `docker compose up --build` (build on a connected machine, `docker save`, carry into the lab).

## Results

**Generated golden set (16):** 16/16 correct, 0 false positives.

**Real-world set (16 files, public datasets):**

| Exhibit | Truth | PRAMAAN | Why |
|---|---|---|---|
| R01 iPhone photo | original | No indications | |
| R02 Canon photo re-saved in GIMP | edited | **Indications** | editor in Software tag, size ≠ EXIF size |
| R03 Nokia GPS photo | original | Inconclusive | consistent, stays cautious |
| R04 same photo, date edited | edited | **Indications** | GPS satellite clock 3 days off |
| R11 street CCTV, DVR export | original | Inconclusive | clock consistent |
| R12 CCTV, 8 s deleted | edited | **Strong** | clock skips 8 s at 0:30; VPF shift + re-encode |
| R13 CCTV, footage looped | edited | **Strong** | repeated footage; clock runs backwards |
| R14 recorded statement | original | No indications | lips in sync (+80 ms) |
| R15 statement, audio re-laid | edited | **Indications** | speech 440 ms behind lips |
| R16 statement, words replaced | edited | **Indications** | lip-sync breaks after ≈14 s |
| R20/R23/R25 genuine recordings | original | Inconclusive | ENF abstains (weak/no hum) |
| R21 room recording, 3.3 s cut | edited | Inconclusive | hard case: hum too noisy, ENF abstains |
| R22 recording with 6 s insert | edited | Inconclusive | noise step found but below threshold |
| R24 AI voice clone | edited | **Indications** | exact digital silence typical of TTS |

**7 of 9 real edits caught, 0 of 7 originals falsely flagged.** Both misses are honest abstentions rather than wrong answers. `tests/test_platform.py` locks these results in.

Real-world sources: ianare/exif-samples, OpenCV `vtest.avi` (PETS-style street CCTV), the GRID audio-visual corpus (via rizkiarm/LipNet), ENF-WHU-Dataset, resemble-ai/Resemblyzer. Edited versions were derived by `scripts/fetch_real_samples.py`. ManTra-Net's web forgeries were deliberately left out because they show real, identifiable people. Check each source's licence before redistributing.

## Detectors

| Requirement | Built-in (CPU, always on) | Deep-learning plug-in (on when weights installed) |
|---|---|---|
| Image & metadata | EXIF/XMP/GPS rules, thumbnail vs image, sun position, editor/AI markers, C2PA, PRNU camera match | **TruFor** (auto-detected in `models/TruFor`, see above), **AI-image classifier** |
| Compression | Quantisation tables, JPEG ghosts, noise map, copy-move, ELA | **CAT-Net v2** (`PRAMAAN_CATNET_CMD`) |
| CCTV integrity | PTS gaps/reversals, keyframe cadence, frozen runs, noise-verified loops, discontinuities | |
| Video re-encoding | **Double-compression / VPF**: earlier GOP size, frame-deletion phase shift | full G-VPF from decoder stats (roadmap) |
| Timestamps | OSD clock OCR vs file clock vs motion | |
| Face deepfakes | face crops + contact sheet | **DeepfakeBench** (`PRAMAAN_DEEPFAKE_CMD`, adapter in `models/`) |
| Cross-modal | **lip-sync offset & mid-clip break** | SyncNet (roadmap) |
| Audio & voice | ENF phase (50/60 Hz, harmonics), noise floor, digital silence, bandwidth, editor tags | LFCC or **SSL+AASIST** spoof curve; **PartialSpoof** segment localiser (`scripts/train_partialspoof.py` or `PRAMAAN_PARTIALSPOOF_CMD`) |

Every plug-in was tested end to end with a stand-in scorer that follows its command contract, so a real model dropped in produces findings, timeline tracks, image layers and report entries. Until weights are installed the check shows **Not installed** and is excluded from fusion rather than guessing.

## Live demo script (5 minutes)

1. **Home page (30 s).** Drag the lamp across the wisteria photo to reveal its error-level map. Click "Verify the whole chain".
2. **Real CCTV (90 s).** Open *R12 Same CCTV with 8 s removed*. The clock line steps at 0:30. Click the red span and show the before/after frames.
3. **Doctored statement (60 s).** Open *R15* and point to the 440 ms offset, then *R16* for the break at ≈14 s.
4. **Honesty (30 s).** Open *R21*: ENF abstains because the hum is too noisy. "We'd rather say inconclusive than accuse."
5. **Court (60 s).** Sign the report (English + हिन्दी), scan the QR code, flip one byte of the PDF, and show that verification now fails.

## Official PS-05 dataset: results

Run the organisers' folder in one command. Every sub-folder becomes a case in the dashboard, and the results
are scored against their description sheets ("Voice clip details.txt", "Video details sheet.txt"):

```bash
python scripts/eval_dataset.py "PS-05.Digital evidence tampering detection system" --out ps05_results
```

**21 of 21 labelled files correct, 0 false alarms.**

| Folder | Truth (organisers) | PRAMAAN | How |
| --- | --- | --- | --- |
| Voice 1–5 | AI-generated | Strong · AI-generated | ElevenLabs **C2PA Content Credentials**: signature and content hash verified offline |
| Voice 6 | real | not flagged | no manifest, no artefacts |
| Image copies 1–5 | tampered | Strong (4) / Indications (1) | camera metadata typed into PNG text chunks, unknown Make "CCSU", alpha channel in a "camera" photo, FFmpeg-encoded "camera" JPEG, AI-native size; **copy ↔ original comparison** (enlarged, alpha added, metadata added, OpenAI C2PA stripped) |
| Image originals 3–5 | (references) | Strong · AI-generated | OpenAI C2PA manifests (ChatGPT gpt-image), verified |
| Mismatched extension | disguised / empty | Indications · Disguised file type; empty file reported | file signature (magic bytes) vs extension; analysed as the real type |
| Video-1 | clipped | Indications · Footage removed | on-screen clock jumps 6 s in 4 s of video |
| Video-2 | clipped + night theme | Indications | colour filter, frames deleted, pillarbox re-framing |
| video-3 | night theme | Indications · Colour filter | colour computed from brightness (tint filter) |
| video-4, video-5 | original | not flagged | DVR frame dropping recognised as normal, not as padding |

Metadata is produced in the exact PS-05 output format (File Name, File Type, Image Width/Height, Date Time
Original, CreateDate, ModifyDate, GPSLatitude/Longitude, Make, Model, Software, Description, Artist, Copyright)
for images, audio, video and documents. It appears in the exhibit's **Metadata** tab (copy / CSV / JSON), in
section 2A of the signed report, and per case via **Metadata CSV**.

**Synthetic voice without Content Credentials:** `python scripts/install_models.py --voice` adds a pretrained
wav2vec2 detector (default `MelodyMachine/Deepfake-audio-detection-V2`, override with `PRAMAAN_VOICE_MODEL`). Its
accuracy on Indian-language and phone audio is not established, so it can raise a verdict to "Indications"
but never decides a "Strong" verdict alone; check it on labelled clips from your own cases first.

## Put it online (Render)

`render.yaml` + `Dockerfile.render` deploy the whole platform, with TruFor and the AI-image model built into the
image, a persistent disk for cases, and a site password. Step-by-step: **[DEPLOY_RENDER.md](DEPLOY_RENDER.md)**.

## Deleting cases and the case log

Cases can be deleted from the Cases list (bin icon) or from inside a case ("Delete case"). You must give a reason and
type the FIR number to confirm. Deletion removes the exhibits, analysis results, heatmaps and reports, but **never the history**:

* **Case log** (`/log/`, table `case_log`): append-only (SQLite triggers block edits and deletes) and hash-chained.
  It records `CASE_CREATED` (all case fields), `EVIDENCE_ADDED` (file name, size, SHA-256, MD5) and `CASE_DELETED`,
  which stores a snapshot of everything the case held (every exhibit's hashes, verdicts and reports) plus who deleted it, when and why.
  It can be searched, filtered and downloaded as JSON. Cases that existed before the log was switched on are recorded at the next start.
* **Custody ledger**: untouched. Each exhibit gets a final `EVIDENCE_DELETED` entry, so the chain still verifies.
* Cases with analyses still running cannot be deleted until they finish. API: `DELETE /api/cases/{id}` with `{"reason": "..."}`; `GET /api/caselog`.

## Limitations (say them before judges do)
- Deep-learning detectors need their weights. Cross-domain accuracy of deepfake and voice detectors is poor in published in-the-wild tests, so those families are capped.
- An OSD jump can be innocent (motion-triggered recording, NTP clock sync); every finding lists these alternatives.
- ENF needs a clean hum. Timestamp *matching* needs the lab's own logger, because Grid-India publishes only 15-minute averages.
- The Section 63 annex is a draft for a qualified human expert to sign.
