<div align="center">

# AromaGen: Interactive Generation of Rich Olfactory Experiences with Multimodal Language Models

**Open-source stack for programmable multi-channel olfactory displays — compose, play, and refine scents from language, speech, and images.**

[![Paper](https://img.shields.io/badge/Paper-2604.01650-B31B1B?style=flat-square)](https://arxiv.org/abs/2604.01650)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)
[![Python 3.9+](https://img.shields.io/badge/Python-3.9%2B-3776AB?style=flat-square)](https://www.python.org/)

[Quick start](#quick-start) · [Repository](#repository-structure) · [API](#api-ai-backend-port-8000) · [Hardware](#hardware) · [Docs](#documentation) · [Citation](#citation)

</div>

**AromaGen** maps free-form text, speech, and images to timed pulse trains over a fixed 6–12 channel odorant palette, with human-in-the-loop refinement. This repo ships the composition backend (FastAPI + multimodal LLM), BLE device server, demo UIs, and study instrumentation templates. Developed at the MIT Media Lab. Production interaction logs, RAG corpora, and participant exports are not included — see [`docs/PUBLIC_RELEASE.md`](docs/PUBLIC_RELEASE.md).

---

## Repository structure

```
AromaGen/
├── aromagen/                 # Core Python package
│   ├── agents/               # FastAPI backend (compose, feedback, accept)
│   ├── data/
│   │   ├── samples/          # Public schema examples only
│   │   ├── dialogue/         # Local logs (gitignored)
│   │   └── exports/          # Training splits (generated locally)
│   ├── ui/demo/              # Main demo (:8080)
│   ├── ui/simplex/           # Anchor simplex UI (:8081)
│   ├── ui/axis/              # Axis mixer UI (:8082)
│   ├── scent_classification.json
│   └── cartridge_sets.json
├── backend.py                # BLE device server (port 5001)
├── scripts/                  # Startup & data export utilities
├── docs/                     # Study protocol & documentation
├── analysis/                 # Demo corpus analysis: themes, noise, scent usage, feedback coding
└── requirements.txt          # BLE backend dependencies
```

---

## Quick start

### Prerequisites

- Python 3.9+
- OpenAI API key
- BLE neck-worn scent device (name must contain `wear`)

### 1. Clone and configure

```bash
git clone https://github.com/AwuChen/AromaGen.git
cd AromaGen
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install -r aromagen/agents/requirements.txt

cp .env.example .env
# Edit .env and set OPENAI_API_KEY
```

### 2. Start all services

```bash
./start_all.sh
```

Or run separately:

| Service | Command | URL |
|---------|---------|-----|
| BLE backend | `./start_ble_backend.sh` | http://localhost:5001 |
| AI backend | `./restart_ai_backend.sh` | http://localhost:8000 |
| Demo UI | `./start_frontend.sh` | http://localhost:8080 |
| Simplex navigation UI | `./scripts/start_simplex_ui.sh` | http://127.0.0.1:8081 |
| Axis mixer UI | `./scripts/start_axis_ui.sh` | http://127.0.0.1:8082 |

### 3. Use the demo

1. Open **http://localhost:8080**
2. Describe a memory or smell experience (voice, text, or image)
3. Play the generated 30-second sequence on the device
4. Give spoken feedback to refine; press **This works** to save for future compose

---

## API (AI backend, port 8000)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/compose` | POST | `{ "sentence": "..." }` → scent sequence |
| `/feedback` | POST | Refine sequence from user feedback |
| `/accept` | POST | Save accepted composition to example bank |
| `/transcribe` | POST | Audio → text (multipart) |
| `/describe_image` | POST | Image → sensory text (multipart) |

Interactive docs: http://localhost:8000/docs

---

## Data & logging (local only)

By default, new sessions can be logged under `aromagen/data/dialogue/` when `DIALOGUE_LOGGING=true`. Those files are **gitignored** and are not published with this repo.

Optional **interaction logging + retrieval** (frontend “Log” button, `interaction_log.jsonl`, embeddings cache) is configured per installation via `.env`. Leave unset for a clean public clone; set `INTERACTION_RETRIEVAL_ENABLED=false` until you have your own log.

Schema reference: [`aromagen/data/samples/dialogue_sample.jsonl`](aromagen/data/samples/dialogue_sample.jsonl). Details: [`aromagen/data/README.md`](aromagen/data/README.md).

**Export training JSONL** (after you collect local sessions):

```bash
python scripts/export_training_data.py
```

---

## Configuration

Environment variables (`.env`, see `.env.example` for the full annotated list):

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_API_KEY` | — | Required |
| `OPENAI_MODEL` | `gpt-4o-mini` | Compose / feedback model |
| `LEARNED_EXAMPLES_ENABLED` | `true` | Inject accepted examples at compose |
| `LEARNED_EXAMPLES_TOP_K` | `3` | Retrieved examples |
| `DIALOGUE_LOGGING` | `true` | Write JSONL logs |
| `SEQUENCE_TOTAL_SECONDS` | `30` | Total sequence duration |
| `DESCRIPTOR_FILTER_TOP_K` | `24` | Odorants shown to the model per request (≥ catalog size = no filtering) |
| `ENSEMBLE_ENABLED` | `true` | Self-consistency ensemble for concrete-smell requests |
| `PULSE_SECONDS` / `PULSE_ROUNDS` | `1` / `8` | Interleaved hardware pulse-train timing |
| `INTERACTION_LOG_APPS_SCRIPT_URL` / `INTERACTION_LOG_TOKEN` | — | **Optional.** Private Sheet + Apps Script you deploy (`aromagen/production_logging/apps_script/`). Controls auto-hide when unset. |
| `INTERACTION_LOG_SHEET_URL` | — | Optional browser link for “Open Log Sheet” (never commit a lab sheet URL). |
| `INTERACTION_RETRIEVAL_ENABLED` | `true` | RAG from local `interaction_log.jsonl` (no-op if empty; use `false` on fresh clones) |
| `VALIDATION_LAYER_ENABLED` | `false` | Extra relevance-validation pass after compose |

---

## For collaborators (intern onboarding)

1. Clone repo, set up `.env`
2. Run `./start_all.sh` and verify http://localhost:8080
3. Read `aromagen/data/README.md` and `docs/PUBLIC_RELEASE.md`
4. Review `docs/STUDY_PROTOCOL.md` for study history (formal study data stays private)
5. BLE issues: run `python scan_devices.py` — device name must include `wear`

**Suggested task split:**

- **Dataset:** stats, clustering, figures from `aromagen/data/dialogue/`
- **Training:** SFT on `aromagen/data/exports/training_*.jsonl`
- **Study:** MFC protocol, IRB instruments, July sessions

---

## Hardware

Interface class (what AromaGen assumes). A programmable multi-channel olfactory display: several fixed odorant channels (typically 6–12), each loaded with one base material (liquid perfume, oil, or solid/slow-release), actuated on demand so software can play timed pulse trains rather than only a single static mix. Actuation is hardware-specific (e.g. piezo atomization, pump, solenoid, thermal); channels must support repeated, relatively high-frequency pulses (on the order of sub-second to few-second bursts, many pulses per trial) so ratios are expressed as duty over time across channels. You can use the lab neck-worn build or any custom dispenser that exposes the same control model (per-channel on/off timing, one or more channels active per step).

- Reference build: 12-slot wearable used in studies; palette spans 12 standardized descriptor categories (pleasant → unpleasant) — see cartridge_sets.json
- BLE discovery: devices with `wear` in the name (`backend.py`)

Multi-device sync (optional): `./start_sync_server.sh` + `SYNC_SERVER_URL=ws://...`

---

## Documentation

- [`docs/PUBLIC_RELEASE.md`](docs/PUBLIC_RELEASE.md) — What must not be published; git history notes
- [`aromagen/data/README.md`](aromagen/data/README.md) — Local data layout & schema
- [`docs/STUDY_PROTOCOL.md`](docs/STUDY_PROTOCOL.md) — CHI user study protocol (MFC evaluation)
- [`docs/SETUP.md`](docs/SETUP.md) — Extended setup & troubleshooting
- [`analysis/README.md`](analysis/README.md) — June demo corpus analysis (themes, noise audit, scent usage, feedback coding)

---

## License

MIT License — see [LICENSE](LICENSE).

---

## Citation

```bibtex
@article{aromagen2026,
  title={AromaGen: Interactive Generation of Rich Olfactory Experiences with Multimodal Language Models},
  author={Chen, Awu and Wang, Silang and Wen, Yunge and Yu, Jianing and Eisa, Zinab and Lopes, Ana Julia Santos and Brooks, Jas and Ishii, Hiroshi and Liang, Paul Pu},
  year={2026},
  eprint={2604.01650},
  archivePrefix={arXiv},
  primaryClass={cs.HC},
  url={https://arxiv.org/abs/2604.01650},
  doi={10.48550/arXiv.2604.01650}
}
```

*(Update when published.)*
