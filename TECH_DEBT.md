# Tech debt

Found in a read-only audit on 2026-10-18 (branch `relax-dependency-pins`, commit `f600339`).
Nothing was executed. Numbers match the audit report; bugs are in `BUGS.md`.
Removing the dead and duplicated code below saves about 1,000 lines and 2 dependencies
(items 15-18 added in a second pass over the model internals).

## 9. Nothing tests that the package still imports after the pins move

- Where: `.github/workflows/install_check.yml`
- Problem: CI only runs `pip install`. The code imports diffusers internals
  (`diffusers.models.lora.LoRACompatibleLinear`, `diffusers.models.attention_processor.Attention`
  in `src/chatterbox/models/s3gen/matcha/transformer.py:5-14`) and transformers logits warpers
  (`src/chatterbox/models/t3/t3.py:13-20`). These are what break first once upper bounds go away,
  and an install-only check stays green when an import fails.
- Fix: one CI step, `python -c "import chatterbox, chatterbox.tts_turbo, chatterbox.vc"`. Keep
  the CPU smoke test from `DEPENDENCY_MIGRATION.md` as a manual check (it needs GBs of weights).
- If skipped: a newer diffusers release breaks every user at import time with green CI.

## 10. Three copies of `Conditionals`, four copies of the device setup

- Where: `src/chatterbox/tts.py:64-103`, `src/chatterbox/mtl_tts.py:113-152`,
  `src/chatterbox/tts_turbo.py:69-108`; MPS fallback and `map_location` logic in `tts.py`,
  `mtl_tts.py`, `tts_turbo.py`, `vc.py`
- Problem: the copies have drifted already. `mtl_tts.py:263` sets
  `t3_cond_prompt_tokens = None` before the `if`; `tts.py` and `tts_turbo.py` do not. A fix in one
  copy misses the others. The three `punc_norm` copies differ on purpose; leave them.
- Fix: keep one `Conditionals` in `tts.py` and import it in the other two (existing
  `from chatterbox.mtl_tts import Conditionals` keeps working). Same for a small
  `_resolve_device(device)` helper.
- If skipped: the next fix lands in one of three places.

## 11. T3 backend rebuilt on every call; dead code around it

- Where: `src/chatterbox/models/t3/t3.py:66`, `:272-301`, `:322`;
  `src/chatterbox/models/t3/inference/t3_hf_backend.py`
- Problem: `self.compiled = False` runs right before `if not self.compiled`, so the cached
  backend is rebuilt every call. Around it: a 16-line commented-out `generate` block, a duplicate
  `top_p_warper` line, unused `deepspeed_patch_applied`, unused parameters
  (`num_return_sequences`, `length_penalty`, `do_sample`, `stop_on_eos`). The default
  `max_new_tokens=None` makes `range(None)` crash for any caller that omits it.
  `prepare_inputs_for_generation` is only used by HF `generate()`, which nothing calls, so the
  backend is a pass-through wrapper.
- Fix: call `self.tfmr` and `self.speech_head` directly, as `inference_turbo` does; delete the
  backend class and the dead lines; default `max_new_tokens` to `self.hp.max_speech_tokens`.
- If skipped: about 150 lines to read for nothing, and a small cost per call.

## 12. Chinese word splitter loads even when Chinese is never used

- Where: `src/chatterbox/models/tokenizers/tokenizer.py:190-197`
- Problem: `pkuseg()` loads a segmentation model in every `MTLTokenizer.__init__`, so every
  multilingual load pays its time and memory even without `zh`. Cost not measured.
- Fix: create it on the first `zh` call.
- If skipped: slower startup and extra memory.

## 13. Small mismatches in the multilingual demo

- Where: `multilingual_app.py:147-148`, `:283`, `:192-206`
- Problem: the `MODEL.to(DEVICE)` branch never runs (the class has no `.to`). The CFG slider
  minimum is 0.2 while the on-screen note says to set CFG to 0. The MCP docstring lists 7
  languages, not 23.
- Fix: delete the branch, set the slider minimum to 0, fix the docstring.
- If skipped: users cannot follow the demo's own advice.

## 14. The `torch.load` patch in `example_for_mac.py` does nothing

- Where: `example_for_mac.py:7-15`
- Problem: the English model loads weights with safetensors, and the one remaining `torch.load`
  (`conds.pt`) already passes `map_location`, so the patch never applies.
- Fix: delete those lines.
- If skipped: people copy a hack they don't need.

## 15. Two dependencies are only needed by dead code (`conformer`, `omegaconf`)

- Where: `src/chatterbox/models/s3gen/matcha/decoder.py:7,161-197`,
  `src/chatterbox/models/s3gen/flow.py:24,57`, `pyproject.toml:25,31`
- Problem: `conformer` is imported only for `ConformerWrapper`, which only `matcha.Decoder` uses,
  which only `matcha.CFM` uses, and nothing creates a `CFM` (see 16). `omegaconf` is used once, to
  wrap a dict in the default value of `decoder_conf`. That value is stored as
  `self.decoder_conf` and never read. Both are still installed and resolved against
  ebook2audiobook. `DEPENDENCY_MIGRATION.md` plans to keep both.
- Fix: delete the dead classes from 16, then replace `DictConfig(` with the `CFM_PARAMS` that
  `flow.py:23` already imports (same values). Remove both from `pyproject.toml` and the migration
  table. `conformer` depends on `einops`, which `perceiver.py` and both decoders import directly,
  so add `einops` to the dependencies in the same change.
- If skipped: two packages to install and keep compatible, with no use.

## 16. About 850 lines of vendored code that nothing calls

- Where (all under `src/chatterbox/models/s3gen/`):
  - `matcha/text_encoder.py`, the whole file (413 lines), is never imported.
  - In `matcha/decoder.py:161-443`, `ConformerWrapper` and `Decoder` are only used by `CFM`.
  - `matcha/flow_matching.py:118-129` defines `CFM`, which nothing creates.
  - `flow_matching.py:63-76` is unreachable: the 14 lines after `raise NotImplementedError` in
    `ConditionalCFM.forward`.
  - `transformer/encoder_layer.py:24-108` defines `TransformerEncoderLayer`, which is never used.
  - `utils/mask.py:19-51` is a commented-out `subsequent_mask`.
  - `hifigan.py:27,163-166,382-394` holds the `remove_weight_norm` methods.
- Problem: this was copied from Matcha-TTS and CosyVoice and never trimmed. The
  `remove_weight_norm` methods are also broken. They call the old `torch.nn.utils.remove_weight_norm`
  on layers wrapped with the new `parametrizations.weight_norm`, which raises an error. They also
  call `self.m_source.remove_weight_norm()`, which does not exist. Nothing calls them (grepped
  repo).
- Fix: delete all of the above. The registries in `utils/class_utils.py` stay, because
  `upsample_encoder.py` selects from them by name.
- If skipped: about 850 lines to read, plus imports of diffusers and conformer that make
  upgrades harder.

## 17. Perceiver uses a torch API that is deprecated, on every generate

- Where: `src/chatterbox/models/t3/modules/perceiver.py:64-73,92-100`
- What: the perceiver condenses the reference-voice tokens. It runs on every call, because
  `use_perceiver_resampler = True` (`t3_config.py:22`) and `flash_attention=True` is the default.
- Problem: `torch.backends.cuda.sdp_kernel` has been deprecated since torch 2.3 in favor of
  `torch.nn.attention.sdpa_kernel`. That comes from the torch docs; it was not run here. The
  flags it sets are all `True`, which is already torch's default, so the context manager does
  nothing.
- Fix: delete the `with` line, `setup_flash_config` and `flash_config`, and call
  `F.scaled_dot_product_attention` directly.
- If skipped: a warning on each call today, and a crash on the torch release that removes the
  API. A loosened torch upper bound will pick that release up.

## 18. `utils/mask.py` calls `logging` without importing it

- Where: `src/chatterbox/models/s3gen/utils/mask.py:17,162`
- Problem: when a mask row is all padding, the warning line raises `NameError` instead of
  warning. Current inference never gets there: it runs one item at a time, and the
  reference-voice tokens are always prepended. A batched or new caller would hit it.
- Fix: add `import logging`.
- If skipped: a confusing crash later, for whoever adds batching.

## Not checked

- Read in full: `hifigan.py`, `decoder.py`, `xvector.py`, `matcha/decoder.py`,
  `matcha/flow_matching.py`, `matcha/transformer.py`, `upsample_encoder.py`, `perceiver.py`,
  `cond_enc.py`, `s3tokenizer/*`, `llama_configs.py`, `voice_encoder/*`, `utils/*`,
  `f0_predictor.py`.
- Only grepped for use: `transformer/attention.py`, `subsampling.py`, `convolution.py`,
  `positionwise_feed_forward.py`, `activation.py` (unchanged vendored WeNet code), and
  `matcha/text_encoder.py` (confirmed unused).
- GPU and MPS paths, and diffusers/transformers versions newer than the current pins.
- Edge cases that were not run: T3 stopping before its first speech token (zero tokens into
  S3Gen), and a reference clip under about 30 ms, which is shorter than the 720-sample
  reflect pad in `mel.py:60`.
