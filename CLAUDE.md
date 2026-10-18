# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- Install (editable): `pip install -e .` (Python >=3.10; README says developed on 3.11). This is the only thing CI runs (`.github/workflows/install_check.yml`, Python 3.10).
- There is **no test suite and no linter config**. Verify changes by running the example scripts at the repo root (`example_tts.py`, `example_tts_turbo.py`, `example_tts_nano.py`, `example_vc.py`, `example_for_mac.py`) or the Gradio demos (`gradio_tts_app.py`, `gradio_tts_turbo_app.py`, `gradio_vc_app.py`, `multilingual_app.py`). Weights download from Hugging Face on first use.
- Quick CPU smoke test: `ChatterboxMultilingualTTS.from_pretrained(device="cpu")`, generate `language_id="en"` and `"fr"`, check output length and finite samples.

## Architecture

Package lives in `src/chatterbox/`. Four public entry-point classes, each in its own top-level module with its own `Conditionals` dataclass and `from_local(ckpt_dir, device)` / `from_pretrained(device)` constructors (`from_pretrained` calls `snapshot_download`, then `from_local`):

- `tts.py` `ChatterboxTTS`: original English model (HF repo `ResembleAI/chatterbox`).
- `mtl_tts.py` `ChatterboxMultilingualTTS`: 23 languages (`SUPPORTED_LANGUAGES`). `t3_model` selects the T3 checkpoint: `"v2"` (default, legacy) or `"v3"`, or any `.safetensors` filename in the checkpoint dir. Resolved by `_resolve_multilingual_t3_model`.
- `tts_turbo.py` `ChatterboxTurboTTS`: Turbo (350M, `GPT2_medium`) **and Nano (110M, `GPT2_small`)**; Nano is the same class with `nano=True`, which switches the repo id, T3 checkpoint filename and llama config. Uses a one-step distilled decoder and paralinguistic tags (`[laugh]` etc.).
- `vc.py` `ChatterboxVC`: voice conversion, reuses S3Gen only.

Only `ChatterboxTTS`, `ChatterboxVC`, `ChatterboxMultilingualTTS` and `SUPPORTED_LANGUAGES` are exported from `__init__.py`; Turbo is imported from `chatterbox.tts_turbo`.

Each TTS pipeline composes the same sub-models from `src/chatterbox/models/`:

- `t3/`: autoregressive text-to-speech-token LLM (HF Llama/GPT2 backbone, `T3Config` variants, conditioning in `T3Cond`/`T3CondEnc`, custom inference backend in `t3/inference/`).
- `s3gen/`: speech tokens to mel to waveform (conformer encoder, flow-matching CFM decoder under `matcha/`, HiFT vocoder). `S3Gen` is shared by TTS and VC.
- `s3tokenizer/`: speech tokenizer (wraps the `s3tokenizer` package) used for reference audio.
- `voice_encoder/`: speaker embedding from the reference clip.
- `tokenizers/`: text tokenizers; multilingual one has per-language preprocessing (Japanese kana, Chinese, Hebrew, etc.).

Cross-cutting behavior to preserve:
- Voice cloning flows through `Conditionals` (built from `audio_prompt_path` via `prepare_conditionals`, or loaded from a `conds.pt` shipped with the checkpoint as the default voice).
- Checkpoints are loaded to CPU first when `device` is `cpu` or `mps` (they may be CUDA-saved); `torch.load` uses `weights_only=True`.
- Every generated waveform is watermarked with `perth.PerthImplicitWatermarker()`. Don't bypass this when changing `generate`.

## Branch context

Branch `relax-dependency-pins` is loosening the exact pins in `pyproject.toml` so the package can coexist with ebook2audiobook. `DEPENDENCY_MIGRATION.md` holds the target pins, constraints and test plan; its status is "not started" and nothing there is verified. Read it before touching dependencies.
