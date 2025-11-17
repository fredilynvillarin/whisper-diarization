# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Common commands

- System deps (required once)
  - macOS: `brew install ffmpeg`
  - Linux (Debian/Ubuntu): `sudo apt update && sudo apt install -y ffmpeg`
  - Windows (Admin PowerShell): `choco install ffmpeg`
  - Cython (build helper): `python -m pip install --upgrade pip cython`

- Install Python dependencies
  - Default (CUDA if available via upstream deps):
    - `pip install -c constraints.txt -r requirements.txt`
  - CPU-only PyTorch (optional):
    - `python -m pip install torch torchaudio --index-url https://download.pytorch.org/whl/cpu`
    - `pip install -c constraints.txt -r requirements.txt`

- Linting and import order (configured via setup.cfg)
  - Install tools: `pip install flake8 isort`
  - Check: `flake8 .` and `isort . --check-only`
  - Format imports: `isort .`

- Tests
  - No test suite is configured. CI performs a smoke run against a sample audio.

## How to run

- README describes a CLI entrypoint: `python diarize.py -a <AUDIO_FILE>` with options like `--whisper-model`, `--device`, etc. If `diarize.py` is present in your checkout, you can use that directly.
- Programmatic usage with the included NeMo-based diarizer:

```python
import torch, torchaudio
from diarization import MSDDDiarizer

# Load audio (expects mono, 16kHz). Resample if needed.
audio, sr = torchaudio.load("tests/assets/test.opus")
if sr != 16000:
    audio = torchaudio.functional.resample(audio, sr, 16000)

model = MSDDDiarizer(device="cuda" if torch.cuda.is_available() else "cpu")
labels = model.diarize(audio)
# labels: List of (start_ms, end_ms, speaker_id)
print(labels[:5])
```

Notes:
- First run will download NeMo models referenced in the config (vad_multilingual_marblenet, titanet_large, diar_msdd_telephonic).
- A sample asset exists at `tests/assets/test.opus` for quick smoke checks.

## Architecture overview

- Core module: `diarization/msdd/msdd.py`
  - `MSDDDiarizer`: thin wrapper around NeMo’s `NeuralDiarizer` using config in `diarization/msdd/diar_infer_telephonic.yaml`.
  - Flow:
    1) Accepts an audio tensor (mono, 16k expected).
    2) Writes a temporary WAV and NeMo manifest.json.
    3) Initializes NeMo diarization components (VAD: MarbleNet, embeddings: TitaNet, clustering, MSDD model) per YAML.
    4) Runs diarization and parses the generated RTTM into tuples `(start_ms, end_ms, speaker_id)`.
- Configuration: `diar_infer_telephonic.yaml`
  - Tuned for 2–8 speakers; uses `vad_multilingual_marblenet`, `titanet_large`, and `diar_msdd_telephonic`.

## CI reference (GitHub Actions)

- Matrix across macOS, Ubuntu, Windows and Python 3.10–3.12: `.github/workflows/test_run.yml`.
- Steps: install ffmpeg; install CPU-only PyTorch; install deps via `uv` with `constraints.txt`; smoke run (if `diarize.py` exists):
  - `python diarize.py -a ./tests/assets/test.opus --whisper-model tiny.en`
