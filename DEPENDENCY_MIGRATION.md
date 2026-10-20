# Dependency pin migration (branch `relax-dependency-pins`)

Goal: loosen `pyproject.toml` pins so `chatterbox-tts` can be installed next to
ebook2audiobook (e2a) without downgrading torch/gradio/numpy for every other engine.

Acceptance bar: standalone chatterbox works as before. The upstream maintainer will not take
regressions for e2a's sake, so every relaxed pin must keep the old pins' versions working (the
"Floors" row under Results). e2a compatibility is the motivation, not a test target.

Status (2026-10-19): **pins relaxed in `pyproject.toml`; CPU matrix complete.** All four
classes pass default-voice and reference-audio tests on Python 3.10, 3.11 and 3.12, with the old
pins (floors), torch 2.6.0, 2.7.1 and 2.14.1, and e2a's `requirements.txt` plus torch 2.7.1
resolves. `resemble-perth` stays on git `@master` (PyPI 1.0.1 is broken, see Results).
Gradio demo apps launched and generated on CPU through the UI API (see Results). GPU backends cannot be
tested on the dev machine (its Tiger Lake iGPU is unsupported by PyTorch XPU).

## Constraints from e2a (`lib/conf.py`, `requirements.txt`)

- Python 3.10 to 3.12 (`min_python_version`, `max_python_version`).
- `torch_matrix` bases are mostly 2.7.1; older backends top out lower
  (cu121 2.5.1, cu124 and rocm6.1 2.6.0, jetson 2.4 to 2.5). Latest is 2.14.1.
- `transformers_caps = {'2.4': '5.1', '2.5': '5.8'}`: torch 2.4 caps transformers at 5.1.
- `gradio==6.29.1` in `requirements.txt`.
- `coqui-tts[languages]==0.27.5`. Its numpy requirement has not been checked.

## Pins and targets

| Dependency | Old pin | Target | Notes |
|---|---|---|---|
| torch, torchaudio | `==2.6.0` (py<3.14) | `>=2.6` | try 2.7.1 first (e2a base); 2.4/2.5 only if cheap, unverified |
| transformers | `==5.2.0` | `>=5.2` | no upper bound unless the smoke test breaks |
| numpy | `<2` (py<3.13) | drop the `<2` cap | likely the main source of fails |
| gradio | `==6.8.0` | removed from `dependencies`; optional extra `demo = ["gradio>=6.8"]` | only the demo apps use it, not the library. Demo apps need `pip install -e ".[demo]"`. Status: done, verified by install check only (no demo app launched) |
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

## Results

CPU smoke test, one short sentence per model (about 2 to 3 s of audio), finite samples
checked. Multilingual (`en`, `fr`) plus Turbo and Nano all passed in every row. Baseline was
the old pins on 3.12 (multilingual only).

| Python | torch | numpy | transformers |
|---|---|---|---|
| 3.12 | 2.6.0 (old pins, baseline) | 1.26.4 | 5.2.0 |
| 3.12 | 2.6.0 | 2.5.4 | 5.20.1 |
| 3.12 | 2.7.1 (multilingual only) | 2.5.4 | 5.20.1 |
| 3.12 | 2.14.1 (torchaudio 2.11.0) | 2.5.4 | 5.20.1 |
| 3.11 | 2.14.1 | 2.4.6 | 5.20.1 |
| 3.10 | 2.14.1 | 2.2.6 | 5.20.1 |

Also resolved: librosa 1.0.0, diffusers 0.41.0, safetensors 0.8.0, gradio 6.30.0.
coqui-tts 0.27.5 only requires `numpy>=1.26.0`, so dropping the `<2` cap does not conflict.
Found by the reference-audio test: `ChatterboxTurboTTS.norm_loudness` upcast the wav to float64 under numpy 2 (`np.float64` gain times a float32 array) and crashed the mel filter step. Fixed with `float(...)` in `tts_turbo.py`; Turbo, Nano and cloning for the other three classes pass after that on 3.12 and 3.10.
Floors: the old pins (numpy 1.26.4, torch 2.6.0, transformers 5.2.0, diffusers 0.29.0, librosa 0.11.0, safetensors 0.5.3, gradio 6.8.0, conformer 0.3.2, pykakasi 2.3.0) were installed together and passed all four classes (multilingual, Turbo, Nano with default voice; TTS, multilingual, Turbo, VC with a reference clip). numpy floor raised from 1.24 to 1.26 because 1.26 is the lowest tested (and coqui-tts needs it).
Reference-audio cloning and VC (all four classes) also pass on 3.11 (torch 2.14.1), 3.10 (torch 2.14.1), 3.12 with torch 2.7.1 (the e2a venv), and the old-pin floors on 3.10 and 3.11 (torch 2.6.0, numpy 1.26.4, transformers 5.2.0), along with the default-voice smoke tests there. The numpy floor of 1.26 is the lowest tested; the original 1.24 floor was never exercised.
`resemble-perth`: keep the git `@master` dependency (1.1.0). The PyPI release 1.0.1 imports `pkg_resources` without declaring setuptools, so in a clean venv `perth.PerthImplicitWatermarker` is `None` and every `generate` crashes with `'NoneType' object is not callable`.
Intel iGPU (Tiger Lake Iris Xe, torch 2.7.1+xpu, 2026-10-19): not usable. `device="xpu"` first fails loading CUDA-saved checkpoints (the `map_location` branch only covers `cpu` and `mps`; a code change, unrelated to the pins). With that bypassed, the first compare-and-sum kernel fails with `RuntimeError: UR error`, in plain torch with no chatterbox code. PyTorch does not support Tiger Lake XPUs at all. GPU remains untested.
The scripts live outside the repo (`~/chatterbox-smoke/`).

e2a venv check (2026-10-19): e2a `requirements.txt` + `torch==2.7.1`/`torchaudio==2.7.1` + this package resolves with `uv pip compile` on 3.12 and 3.10 (3.12: numpy 2.5.4, transformers 5.20.2, librosa 1.0.0, gradio 6.29.1, coqui-tts 0.27.5; 3.10: numpy 2.2.6, librosa 0.11.0). Installed on 3.12 (CPU smoke test: multilingual en/fr, Turbo, Nano all pass; `pip check` clean). e2a installs torch separately via its `torch_matrix`, so it is not in its `requirements.txt`; 2.7.1 is its base.

Gradio demos (2026-10-19, CPU, Python 3.12): `gradio_tts_app`, `gradio_tts_turbo_app`, `gradio_vc_app` and `multilingual_app` each launched and generated via `gradio_client` with a reference clip (TTS, Turbo, VC; multilingual `en` and `fr`; finite output, 2 to 3 s, VC 12 s). Passed with the relaxed pins (gradio 6.30.0, torch 2.14.1) and the old-pin floors (gradio 6.8.0, torch 2.6.0). Launched with `share=False` and `mcp_server` dropped; one app at a time under a 9G cgroup limit. Turbo app logs a harmless `gr.Blocks(css=...)` deprecation warning (Gradio 6). Browser UI not clicked through.

## Not covered

- GPU backends (CUDA, ROCm, XPU, MPS). The dev machine has only a Tiger Lake iGPU, which fails on XPU (see Results).
- Audio quality (only length and finite samples are checked) and clicking through the demo UIs in a browser.
- torch 2.4 and 2.5 (the `>=2.6` floor excludes them).
