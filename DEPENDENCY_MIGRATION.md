# Dependency pin migration (branch `relax-dependency-pins`)

Goal: loosen `pyproject.toml` pins so `chatterbox-tts` can be installed next to
ebook2audiobook (e2a) without downgrading torch/gradio/numpy for every other engine.

Acceptance bar: standalone chatterbox works as before. The upstream maintainer will not take
regressions for e2a's sake, so every relaxed pin must keep the old pins' versions working (the
"Floors" row under Results). e2a compatibility is the motivation, not a test target.

Status (2026-10-20): **pins relaxed in `pyproject.toml`; CPU matrix complete.** All four
classes pass default-voice and reference-audio tests on Python 3.10, 3.11 and 3.12, with the old
pins (floors), torch 2.6.0, 2.7.1 and 2.14.1, and e2a's `requirements.txt` plus torch 2.7.1
resolves. `resemble-perth` stays on git `@master` (PyPI 1.0.1 is broken, see Results).
Gradio demo apps launched and generated on CPU through the UI API (see Results). Nine non-English
languages and a 20 s English text match the old pins (Whisper transcripts, see Results); the torch
floor stays at 2.6 because torch 2.5 needs an older diffusers. GPU backends cannot be
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
| gradio | `==6.8.0` | removed from `dependencies`; optional extra `demo = ["gradio>=6.8"]` | only the demo apps use it, not the library. Demo apps need `pip install -e ".[demo]"`. Status: done; all four demo apps launched and generated on CPU (see Results) |
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
4. Repeat on Python 3.10 and 3.11 (done, see Results).

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

Audio quality (2026-10-19, CPU, Python 3.12, seed 0, one 80-character English sentence; old pins = torch 2.6.0 / numpy 1.26.4 / transformers 5.2.0, relaxed = torch 2.14.1 / numpy 2.5.4 / transformers 5.20.1): TTS, multilingual `en`, Turbo and Nano with the default voice, run through Whisper-base. All eight clips transcribe with 0% WER against the input text. Loudness (RMS within 1.3 dB per model), spectral centroid (within 400 Hz) and high-band (>8 kHz) energy are the same between old and relaxed pins, with under 0.01% of samples at |x| >= 0.999. Durations differ by up to 0.8 s (sampling is not reproducible across torch versions, so clips are not sample-aligned). VC is deterministic: old and relaxed output match to 98 dB SNR (max abs diff 2e-4). TTS peaks slightly above 1.0 (1.01 old, 0.97 relaxed) and VC to 1.07 in both, so the old pins already did this. Not checked: non-English languages, long text, subjective listening.
Regression found while running this, fixed: `torchaudio.save` raises `ImportError: TorchCodec is required` on torchaudio 2.11 (relaxed pins) without `torchcodec`, even with `backend="soundfile"`. It works on 2.6.0 (old pins) and 2.7.1 (e2a); the break first appears in torchaudio 2.9.0. `~/chatterbox-smoke/tasave.py` (a one-second `ta.save(..., backend="soundfile")`) succeeds on 2.8.0 and raises the same ImportError on 2.9.0 and 2.10.0, each in a fresh CPU-only venv with matching torch. The library never calls it, but every `example_*.py` and the README snippets did. They now use `soundfile.write(..., subtype="FLOAT")` (soundfile comes with librosa; FLOAT is what `ta.save` wrote, so peaks above 1.0 are not clipped). Checked on torchaudio 2.6.0, 2.7.1 and 2.11.0: the written file is FLOAT and reads back sample-identical. On 2.6.0, `ta.save` and the new `sf.write` produce byte-identical files (both go through libsndfile). All five `example_*.py` scripts run end to end on CPU in the relaxed venv (Turbo and Nano with `device="cpu"`, the reference clip as `YOUR_FILE.wav`) and write finite FLOAT 24 kHz WAVs. The Gradio apps and examples do not call `ta.load`, and the apps do not call `ta.save`.

Non-English and long text (2026-10-20, CPU, Python 3.12, multilingual model, `~/chatterbox-smoke/gen_lang.py`): `fr de es ru ja zh ar hi he` (one short sentence each) plus a 6-sentence English text (about 20 s) ran with the old pins (torch 2.6.0 / numpy 1.26.4) and the relaxed pins (torch 2.14.1 / numpy 2.5.4); every clip is finite. Sentences run 2.6 to 4.3 s, and per-sentence RMS differs by at most 0.02 between pin sets (Hebrew; the rest within 0.01). The long text runs 19.5 s (old) vs 20.0 s (relaxed), RMS 0.137 vs 0.135. Transcripts from faster-whisper `small` (`~/chatterbox-smoke/transcribe.py`): `de es fr ja zh hi ru ar` are intelligible under both pin sets, with minor word errors of the same kind. With seed 0, relaxed `de` and `he` looked worse, so `gen_seeds.py` reran both with seeds 1 to 4 under each pin set. `de` came out right or nearly right in 2 of 4 runs with the old pins and 4 of 4 with the relaxed pins. `he` was poor in every run under both. The seed-0 gap is sampling variation, not a regression. The long English text transcribed as one sentence under both sets (Whisper collapses the repeats; the duration shows all six were spoken). Not checked: listening, other languages, texts longer than 20 s.
Lower torch floor (2026-10-20, `uv` venv, Python 3.12, torch 2.5.1, transformers 5.8.1, numpy 2.5.4, `smoke.py`): fails with the diffusers that resolves today (0.41.0): `AttributeError: module 'torch' has no attribute 'accelerator'` in `diffusers/utils/torch_utils.py`. With `diffusers==0.35.2` and `0.32.2` the multilingual `en` and `fr` smoke test passes. The first failing diffusers release lies somewhere after 0.35.2 and no later than 0.41.0; it was not bisected. `pyproject.toml` cannot make a diffusers cap depend on the torch version, so lowering the floor to 2.5 would mean capping diffusers for every user. Decision: keep `torch>=2.6`. As a result, chatterbox cannot be installed on e2a's cu121 and jetson backends (torch 2.4 and 2.5). torch 2.5.1 ran only with the floor bypassed (`--no-deps`) and an older diffusers. torch 2.4 is untested. Turbo, Nano and cloning were not run on 2.5.1. The 2.5.1 venv was temporary; rebuild it from the versions above.
e2a pipeline: e2a `main` (0a98ac31) has no chatterbox engine (`grep -rli chatterbox --include=*.py --include=*.txt --include=*.toml` finds nothing; its README lists chatterbox as an unchecked TODO), so there is no pipeline run to do beyond the venv check above.

## Not covered

- GPU backends (CUDA, ROCm, XPU, MPS). The dev machine has only a Tiger Lake iGPU, which fails on XPU (see Results).
- Listening to the audio (checks above are objective only) and clicking through the demo UIs in a browser (the demos were driven through `gradio_client` only).
- torch 2.4 (the `>=2.6` floor excludes it); torch 2.5 only partly (see Results).
