# Dependency pin migration (branch `relax-dependency-pins`)

Goal: loosen `pyproject.toml` pins so `chatterbox-tts` can be installed next to
ebook2audiobook (e2a) without downgrading torch/gradio/numpy for every other engine.

Status: **not started**. Nothing below has been tested yet. "Target" is the
intent, not a verified result.

## Constraints from e2a (`lib/conf.py`, `requirements.txt`)

- Python 3.10 to 3.12 (`min_python_version`, `max_python_version`).
- `torch_matrix` bases are mostly 2.7.1; older backends top out lower
  (cu121 2.5.1, cu124 and rocm6.1 2.6.0, jetson 2.4 to 2.5). Latest is 2.14.1.
- `transformers_caps = {'2.4': '5.1', '2.5': '5.8'}`: torch 2.4 caps transformers at 5.1.
- `gradio==6.29.1` in `requirements.txt`.
- `coqui-tts[languages]==0.27.5`. Its numpy requirement has not been checked.

## Pins and targets

| Dependency | Current pin | Target | Notes |
|---|---|---|---|
| torch, torchaudio | `==2.6.0` (py<3.14) | `>=2.6` | try 2.7.1 first (e2a base); 2.4/2.5 only if cheap, unverified |
| transformers | `==5.2.0` | `>=5.2` | no upper bound unless the smoke test breaks |
| numpy | `<2` (py<3.13) | drop the `<2` cap | likely the main source of fails |
| gradio | `==6.8.0` | `>=6.8`, or move to an optional extra | only the demo apps use it, not the library |
| librosa | `==0.11.0` | `>=0.11` | check numpy 2 support |
| safetensors | `==0.5.3` | `>=0.5.3` | |
| diffusers | `==0.29.0` | `>=0.29` | pinned old; check against transformers 5.x |
| conformer | `==0.3.2` | `>=0.3.2` | |
| pykakasi | `==2.3.0` | `>=2.3` | e2a leaves it unpinned |
| resemble-perth | git `@master` | PyPI release if one exists | git dependency blocks wheels and reproducible installs |
| s3tokenizer, spacy-pkuseg, pyloudnorm, omegaconf | unpinned | unchanged | |

## Test plan

No test suite exists in the repo. Use a CPU smoke test (load
`ChatterboxMultilingualTTS`, generate `en` and `fr`, check length and finite samples).

1. Baseline: unmodified pins, Python 3.12, CPU.
2. Relaxed pins, same Python and smoke test, torch 2.7.1 then newest.
3. Fix whatever fails (expect numpy 2 and transformers 5.x API drift).
4. Repeat on Python 3.10 and 3.11 if time allows.

## Not covered

- GPU backends (CUDA, ROCm, XPU, MPS). No GPU on the dev machine.
- Turbo and Nano variants, unless the smoke test is extended.
- Whether e2a's own venv resolves with these loosened pins.
