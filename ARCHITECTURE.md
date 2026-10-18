# Architecture

Written 2026-10-18 against `25385bf` (branch `relax-dependency-pins`). Static read only: nothing was run,
no weights were downloaded. Dependency pins are out of scope (`DEPENDENCY_MIGRATION.md`). Known defects are
in `BUGS.md` (cited as **B#**) and `TECH_DEBT.md` (cited as **T#**); this file does not re-audit them.

How to read it:

- **decision**: consistent across sites, with a reason visible in a comment, commit message or constraint.
  Where no reason is recorded the entry says "reason not recorded". Do not invent one.
- **drift**: the pipelines disagree and no reason is visible. "Newest" comes from `git blame` per line, not per
  file. "Newest wins" is only ever a **hypothesis**; row D6 below is a counterexample.
- **vendored**: upstream code. It does not set house style.
- **inferred**: derived by reading, with no comment, commit message or test backing it. Treat as a question.
- Paths: `tts.py`, `mtl_tts.py`, `tts_turbo.py`, `vc.py` are `src/chatterbox/*.py`. `t3.py` is
  `src/chatterbox/models/t3/t3.py`. Other model files are written with their subpackage (`s3gen/s3gen.py`).
  Root scripts are written by name.

## 0. Corrections to CLAUDE.md

CLAUDE.md is untracked and this task may not change it, so the corrections live here. Each was re-read in code.

1. **VC has no `Conditionals`.** It keeps only the S3Gen half, as a plain `ref_dict` (`vc.py:24-36`, `vc.py:48-51`).
2. **`from_pretrained` does not always call `snapshot_download`.** `tts.py:177-180` and `vc.py:71-74` loop over
   `hf_hub_download` and take the parent of the last path. Only `mtl_tts.py:242` and `tts_turbo.py:208` use
   `snapshot_download`.
3. **`weights_only=True` has an exception**: `vc.py:50` (B8). The other six `torch.load` calls have it (D6).
4. **CPU-first `map_location` only reaches `torch.load` calls.** In `tts.py`, `tts_turbo.py` and `vc.py` that is only
   `conds.pt` (`tts.py:163`, `tts_turbo.py:186`, `vc.py:50`). The `load_file` (safetensors) calls take no device
   (`tts.py:140,145,153`, `tts_turbo.py:147,163,171`, `vc.py:55`). That they land on CPU is a library default
   (inferred), not something this repo states.
5. **Turbo's tokenizer is typed as `EnTokenizer`** (`tts_turbo.py:120`) **but is an HF `AutoTokenizer`**
   (`tts_turbo.py:177`). Never infer behaviour from that annotation.
6. **"`T3Config` variants" is only half true.** `T3Config.english_only()` / `.multilingual()` exist
   (`t3_config.py:33-41`), but Turbo/Nano build theirs inline by mutating attributes (`tts_turbo.py:153-159`).
7. **The custom backend in `t3/inference/` serves only the Llama path.** `T3.inference` builds it
   (`t3.py:277-284`); `inference_turbo` calls `self.tfmr` directly (`t3.py:417`). The alignment analyzer that used to
   live there was deleted in `3f35dfc`.
8. **The three `Conditionals` dataclasses are byte-identical** (`tts.py:64-103` = `mtl_tts.py:113-152` =
   `tts_turbo.py:69-108`, checked with `diff`). T10 says "the copies have drifted"; the drift is in
   `prepare_conditionals` (R7), not in the dataclass. The four MPS-fallback blocks are also identical
   (`tts.py:170-175`, `mtl_tts.py:234-239`, `tts_turbo.py:194-199`, `vc.py:64-69`).

Verified as stated: four public classes; `ChatterboxTTS`, `ChatterboxVC`, `ChatterboxMultilingualTTS` and
`SUPPORTED_LANGUAGES` are the only exports (`__init__.py:9-11`); Nano switches repo id, T3 file and backbone config
(`tts_turbo.py:154,162,202`); `t3_model` is resolved by `_resolve_multilingual_t3_model` (`mtl_tts.py:58-68`);
every waveform is watermarked (D2). Not verifiable offline: "weights download on first use".

## 1. Layering

Direction of dependency, verified by `grep -rnE 'from \.\.\.|chatterbox\.(tts|mtl_tts|tts_turbo|vc)' src/chatterbox/models`
(no hits: nothing under `models/` imports a pipeline):

```
root scripts (examples, gradio apps)
   -> src/chatterbox/{tts, mtl_tts, tts_turbo, vc}.py      (pipelines; each is self-contained)
        -> models/t3            -> models/utils.py (AttrDict)
        -> models/s3gen         -> models/s3tokenizer, models/utils.py
        -> models/s3tokenizer   -> pip `s3tokenizer`
        -> models/tokenizers    (no repo imports; HF hub, `tokenizers`)
        -> models/voice_encoder (no repo imports)
```

The only cross-subpackage edges inside `models/` are `s3gen/s3gen.py:23` (to `s3tokenizer`) and
`t3/t3.py:27`, `s3gen/configs.py:1` (to `models/utils.py`). Pipelines import each other not at all; each carries
its own copies of `Conditionals`, `punc_norm`, the MPS check and the CPU-first logic (D1).
VC uses `S3Gen` only (`vc.py:9-10`).

Tags: **[R]** read in full or nearly; **[H]** header, imports and class list only; **V-hdr** vendored with an upstream
header; **V-inf** vendored, inferred; **1P** first-party (a Resemble header, or no upstream header, inferred).

### Pipelines (`src/chatterbox/`)

- `__init__.py` [R]: exports TTS, VC, MTL and `SUPPORTED_LANGUAGES`; reads the version from package metadata. Turbo not exported.
- `tts.py` [R]: `ChatterboxTTS`, English, Llama 520M T3 (`t3_config.py:16`, via `T3()` at `tts.py:144`), 10-step S3Gen
  (`s3gen/s3gen.py:313`). HF repo `ResembleAI/chatterbox` (`tts.py:19`).
- `mtl_tts.py` [R]: `ChatterboxMultilingualTTS`, 23 languages, `t3_model` selector. Same HF repo as `tts.py`.
- `tts_turbo.py` [R]: `ChatterboxTurboTTS`, Turbo (GPT2-medium) and Nano (GPT2-small via `nano=True`), meanflow S3Gen (`tts_turbo.py:170`), 2 steps (`tts_turbo.py:313-317`).
- `vc.py` [R]: `ChatterboxVC`, S3Gen only.

### `models/`

- `utils.py` [R]: `AttrDict`. Imported by `t3.py` and `s3gen/configs.py`.
- `tokenizers/tokenizer.py` [R] 1P: `EnTokenizer`, `MTLTokenizer`, per-language preprocessing, `ChineseCangjieConverter`. `__init__.py` re-exports both classes.
- `voice_encoder/voice_encoder.py` [R: head and defs] V-hdr (`voice_encoder.py:1-2`, Real-Time-Voice-Cloning, MIT): `VoiceEncoder`, speaker embedding. `config.py` [R] and `melspec.py` [H] are 1P/unknown (no header). `__init__.py` re-exports.
- `s3tokenizer/s3tokenizer.py` [R: first 60 lines]: subclass of pip `s3tokenizer.model_v2.S3TokenizerV2` (`s3tokenizer.py:8-22`), a wrapper, not a copy. `__init__.py` [R] holds `SOS`/`EOS` and `drop_invalid_tokens`.
- `t3/__init__.py` [R]: exports `T3`.
- `t3/t3.py` [R] 1P (`t3.py:1-2`, Resemble MIT): the AR text-to-speech-token model. `inference` (Llama path) and `inference_turbo` (GPT2 path).
- `t3/llama_configs.py` [R] 1P: `LLAMA_CONFIGS` registry (`Llama_520M`, `GPT2_medium`, `GPT2_small`).
- `t3/modules/t3_config.py` [R] 1P: `T3Config` hyperparameters; imports `llama_configs`.
- `t3/modules/cond_enc.py` [R] 1P: `T3Cond` dataclass and `T3CondEnc`; imports `perceiver`, `t3_config`.
- `t3/modules/perceiver.py` [H] 1P (`perceiver.py:1-3`, Resemble MIT): `Perceiver` resampler for the speech prompt.
- `t3/modules/learned_pos_emb.py` [R] 1P: `LearnedPositionEmbeddings`.
- `t3/inference/t3_hf_backend.py` [H] 1P: `T3HuggingfaceBackend`, a `LlamaPreTrainedModel` wrapper (T11).
- `s3gen/__init__.py` [R]: aliases `S3Token2Wav` as `S3Gen`; exports `S3GEN_SR`.
- `s3gen/s3gen.py` [R] V-hdr, house-modified (`s3gen.py:1`, "Modified from CosyVoice"): `S3Token2Mel`, `S3Token2Wav`. The glue every pipeline uses; imports all other `s3gen/*`.
- `s3gen/const.py` [R], `s3gen/configs.py` [R] (`CFM_PARAMS`): 1P constants.
- `s3gen/flow.py` [R: parts] V-hdr (Alibaba, Apache-2.0), rewritten in `bc58894`: `CausalMaskedDiffWithXvec`. Imports `utils/mask`, `configs`.
- `s3gen/flow_matching.py` [R: grep] V-hdr, modified in `bc58894`: `ConditionalCFM`, `CausalConditionalCFM`. Imports `matcha/flow_matching`, `configs`.
- `s3gen/decoder.py` [H] V-hdr, modified in `bc58894`: `ConditionalDecoder`. Imports `matcha/decoder`, `matcha/transformer`, `utils/mask`, `utils/intmeanflow`.
- `s3gen/hifigan.py` [H] V-hdr (`hifigan.py:1-5`, "adapted from CosyVoice"): `HiFTGenerator`. No repo imports.
- `s3gen/f0_predictor.py` [H] V-hdr (Alibaba): `ConvRNNF0Predictor`. No repo imports.
- `s3gen/xvector.py` [H] V-hdr (`xvector.py:3-5`, FunASR MIT, "modified from 3D-Speaker"): `CAMPPlus` speaker encoder. No repo imports.
- `s3gen/matcha/decoder.py`, `flow_matching.py`, `transformer.py` [H] **V-inf**: no header. Evidence is `utils/mel.py:1` ("mel-spectrogram extraction in Matcha-TTS") and T16, not the directory name. They import `diffusers` and `conformer`.
- `s3gen/matcha/text_encoder.py` [H] V-hdr (`text_encoder.py:1`, "from jaywalnut310/glow-tts"): never imported (T16).
- `s3gen/transformer/__init__.py`: empty. The eight files below are [H], all V-hdr with Apache-2.0 headers; "WeNet" and "ESPnet" are inferred from the named copyright holders. Described by provenance and importer only.
- `s3gen/transformer/activation.py`: header Johns Hopkins / Shinji Watanabe, 2020 (`activation.py:1`). Imported by `utils/class_utils.py:17`.
- `s3gen/transformer/attention.py`: header Shigeki Karita, 2019 (`attention.py:1`). Imported by `utils/class_utils.py:32`.
- `s3gen/transformer/convolution.py`: header Mobvoi, 2020 (`convolution.py:1`). Imported by `upsample_encoder.py:24`.
- `s3gen/transformer/embedding.py`: header Mobvoi, 2020 (`embedding.py:1`). Imported by `utils/class_utils.py:26`.
- `s3gen/transformer/encoder_layer.py`: header Mobvoi, 2021 (`encoder_layer.py:1`). Imported by `upsample_encoder.py:25`; `TransformerEncoderLayer` inside is unused (T16).
- `s3gen/transformer/positionwise_feed_forward.py`: header Shigeki Karita, 2019 (`positionwise_feed_forward.py:1`). Imported by `upsample_encoder.py:26`.
- `s3gen/transformer/subsampling.py`: header Mobvoi, 2021 (`subsampling.py:1`). Imported by `utils/class_utils.py:18`.
- `s3gen/transformer/upsample_encoder.py`: header Mobvoi, 2021 (`upsample_encoder.py:1`). `UpsampleConformerEncoder` is built at `s3gen/s3gen.py:30,64`; also imports `utils/class_utils` and `utils/mask`.
- `s3gen/utils/class_utils.py` [H] V-hdr (Tsinghua / Xingchen Song): `COSYVOICE_*_CLASSES` registries, selected by name from `upsample_encoder.py:27`.
- `s3gen/utils/mask.py` [H] V-hdr (Karita): `make_pad_mask`, chunk masks. Imported by `flow.py:22`, `s3gen/decoder.py:19`, `upsample_encoder.py:33`.
- `s3gen/utils/mel.py` [H] V-inf ("in Matcha-TTS"): `mel_spectrogram`. Imported by `s3gen.py:27`.
- `s3gen/utils/intmeanflow.py` [R] 1P (inferred: added in `bc58894`, cites arXiv 2510.07979): meanflow time mixer. Imported by `s3gen/decoder.py:23`.
- `s3gen/matcha/`, `s3gen/utils/`, `t3/inference/` and `t3/modules/` have no `__init__.py`; they work as implicit namespace packages (inferred).

### Root consumers (grep of imports and calls only; **not read in full**)

- `example_tts.py` (TTS and MTL), `example_tts_turbo.py`, `example_tts_nano.py` (Turbo with `nano=True`), `example_vc.py`, `example_for_mac.py` (TTS; carries the `torch.load` patch of T14).
- `gradio_tts_app.py` (TTS), `gradio_tts_turbo_app.py` (Turbo; holds the only in-repo paralinguistic tag list, `:9-12`), `gradio_vc_app.py`, `multilingual_app.py` (MTL; `LANGUAGE_CONFIG` at `:16`, `T3_MODEL` env at `:9`).
- The CI in `.github/workflows/install_check.yml` runs `pip install -e .` and nothing else. There are no tests.

## 1.5 Data flow and patterns

### One `generate` call, English pipeline (`tts.py`)

```
reference clip --librosa.load @24k (tts.py:184)--+--> [:DEC_COND_LEN] -> s3gen.embed_ref -> gen dict (tts.py:188-189)
                                                 |      {prompt_token, prompt_token_len, prompt_feat,
                                                 |       prompt_feat_len, embedding}  (s3gen/s3gen.py:165-171)
                                                 +--> resample 16k (tts.py:186)
                                                        +-> [:ENC_COND_LEN] -> S3Tokenizer -> cond_prompt_speech_tokens (tts.py:192-195)
                                                        +-> VoiceEncoder -> speaker_emb, 256-d (tts.py:198-199)
                                                        exaggeration -> emotion_adv (tts.py:204)
                                          T3Cond(t3) + gen dict = Conditionals (tts.py:206)

text -> punc_norm (tts.py:234) -> tokenizer (235) -> duplicate for CFG (237-238) -> pad SOT/EOT (240-243)
     -> T3.inference (tts.py:246): cond_enc concat [speaker, clap(empty), perceiver(prompt), emotion]
        (cond_enc.py:91-96) + text + BOS, then a KV-cached sampling loop with CFG, repetition penalty,
        min_p, top_p (t3.py:338-368); stops on stop_speech_token (t3.py:366)
     -> speech_tokens[0] (tts.py:257) -> drop_invalid_tokens, strips SOS/EOS (s3tokenizer/__init__.py:16-30)
     -> filter < 6561 (tts.py:262)
     -> S3Gen.inference (s3gen/s3gen.py:330): prompt_token ++ tokens (flow.py:161) -> encoder -> CFM decoder
        (10 steps, or 2 with meanflow, s3gen.py:313) -> mel, prompt part cut off (flow.py:196)
        -> HiFT vocoder (s3gen.py:357) -> fade-in of the first samples (s3gen.py:360)
     -> numpy -> [MTL only: trim last token, mtl_tts.py:348-352] -> watermark (tts.py:271) -> tensor (1, T) @ 24 kHz
```

Variants of the same flow:

- **Multilingual:** adds `language_id` (validated against `SUPPORTED_LANGUAGES`, `mtl_tts.py:292-298`) and the `[xx]` token
  (`tokenizer.py:300-302`). No `< 6561` filter (R6).
- **Turbo/Nano:** HF tokenizer, no SOT/EOT padding, no CFG; `T3.inference_turbo` (`tts_turbo.py:298`), `S3Gen(meanflow=True)`
  with 2 steps (`tts_turbo.py:313-317`), 3 silence tokens appended (`tts_turbo.py:310-311`). The reference clip is
  loudness-normalised first (`tts_turbo.py:247-248`).
- **VC:** input audio at 16 kHz goes through `s3gen.tokenizer` (`vc.py:94-97`), then `S3Gen.inference` with the stored target
  voice `ref_dict` (`vc.py:98-101`). No T3, no text.

### Values fixed by the trained weights (change one and the checkpoint stops matching)

All inferred from how the code sizes its tensors; no file states them as a contract.

- Rates and sizes: S3 tokens at 25/s (`s3tokenizer.py:18`); two mel frames per token, enforced with a warning at
  `s3gen/s3gen.py:157-163`; 16 kHz for T3 and VE inputs, 24 kHz for S3Gen (`s3tokenizer.py:15`, `s3gen/const.py:1`).
- T3 inputs: concat order of the conditioning (`cond_enc.py:91-96`); `speech_cond_prompt_len` (see Hidden couplings);
  `text_tokens_dict_size` and the speech vocab 6561 + start/stop (`t3_config.py:11-13`).
- `speaker_embed_size` 256 on both T3 and VE sides.

### Patterns the code uses

- **Orchestrator plus pure modules.** Each pipeline is a plain class (`tts.py:106`, `mtl_tts.py:155`, `tts_turbo.py:111`,
  `vc.py:16`) with no base class and no shared helper module. It wires `nn.Module`s from `models/` and owns I/O,
  device handling and the watermark (D1-D5). Modules do almost no I/O. The exceptions seen are the tokenizers
  (`tokenizer.py:22,170`: vocab file and a Cangjie download) and `cond_enc.py:37`.
- **Copy per pipeline, not inheritance.** `Conditionals`, `punc_norm`, the MPS check and the CPU-first logic are
  copied (§0.8). This is the source of the drift in §3. T10 proposes sharing; nothing has done it yet.
- **Factory class methods** (`from_local`, `from_pretrained`) for construction (D1).
- **Name-to-config registries:** `LLAMA_CONFIGS` (`llama_configs.py:105-109`), `MULTILINGUAL_T3_MODELS`
  (`mtl_tts.py:23-28`). Vendored: `COSYVOICE_*_CLASSES` (`s3gen/utils/class_utils.py`), selected by string from
  `upsample_encoder.py:27`. A new variant is a new dict entry first (D11, §4b-c).
- **Variant selection by flag, not subclass:** `nano: bool` (`tts_turbo.py:136`), `meanflow: bool` (`s3gen/s3gen.py:53,241`),
  `t3_model: str` (`mtl_tts.py:187`), `hp` attributes (`tts_turbo.py:153-159`). The `is_gpt` branches in `t3.py:98,113` keep
  two backbone families in one class.
- **One conditioning object** passed to the model, built from the reference clip (D7).
- **Lazy optional dependencies** with a module-level sentinel (D13).
- **Inference guard:** `torch.inference_mode()` is a `with` block in the pipelines (`tts.py:245`, `mtl_tts.py:324`,
  `vc.py:93`) and a decorator on `T3.inference`, `inference_turbo` and `S3Gen.inference` (`t3.py:225,392`,
  `s3gen/s3gen.py:329`). Turbo's `generate` has no outer guard (`tts_turbo.py:272-320`), so it relies on the decorators.
- **Instance state:** `generate` mutates `self.conds` (`tts.py:220,227`). One caller at a time per instance (B2, B5).

## 2. Decisions

**D1. Every pipeline is one top-level module with the same shape.** It has a class with `sr`, `device` and a
`watermarker`, plus `from_local(ckpt_dir, device, ...)` and `from_pretrained(device, ...)` class methods.
Evidence: `tts.py:110-126,129,168`; `mtl_tts.py:159-175,183,228`; `tts_turbo.py:115-133,136,192`;
`vc.py:20-29,39,62`. Reason not recorded. Exceptions: the variant argument (`t3_model` in MTL, `nano` in Turbo);
VC has no tokenizer or T3.

**D2. Every returned waveform goes through `PerthImplicitWatermarker`, created in `__init__` and applied last.**
Evidence: `tts.py:126,271`; `mtl_tts.py:175,354`; `tts_turbo.py:133,319`; `vc.py:29,103`.
Reason: `README.md:173-175` states every generated file carries the Perth watermark. Exceptions: none. Any new
post-processing (the MTL trim at `mtl_tts.py:348-352`) happens before the watermark call.

**D3. Models never import pipelines; subpackages under `models/` are peers.** Evidence: pipelines import models
(`tts.py:11-16`, `mtl_tts.py:12-18`, `tts_turbo.py:15-22`, `vc.py:9-10`); the only peer edges are `s3gen.py:23` and
`t3.py:27`; the grep above finds no upward import. Reason: not recorded; VC reusing S3Gen alone (`vc.py:9-10,53`)
is consistent with it (inferred). Exceptions: none.

**D4. On `cpu` and `mps`, `torch.load` goes through `map_location=cpu`; otherwise `None`.** Evidence: `tts.py:132-136`;
`mtl_tts.py:192-196`; `tts_turbo.py:139-143`; `vc.py:42-46`. Reason: the comment "to handle CUDA-saved models" at each
site, and `c883726` (fixes issues 351 and 357). **Broken once in-repo**: MTL was cloned in `a9742ee` (2025-09-04)
without it and only got it in `c883726` (2026-03-10). A new pipeline must carry it. Exceptions: safetensors calls take
no device (§0.4).

**D5. `from_pretrained` falls back from `mps` to `cpu` when MPS is unavailable.** Evidence: `tts.py:170-175`;
`mtl_tts.py:234-239`; `tts_turbo.py:194-199`; `vc.py:64-69` (identical text). Reason: `df73ac8` "Attempt to support
cpu/mps". Exceptions: `from_local` has no such check (inferred: an `mps` request on a non-MPS box fails there).

**D6. Checkpoint pickles load with `weights_only=True`.** Evidence: `tts.py:102`; `mtl_tts.py:151,200,213`;
`tts_turbo.py:107`; `cond_enc.py:37`. Reason not recorded (B8 gives the security argument). **Exception: `vc.py:50`.**
A new loader must not copy `vc.py`.

**D7. Voice cloning flows through one conditioning object, built from a reference clip or loaded from `conds.pt`.**
`prepare_conditionals(wav, exaggeration)` builds `Conditionals(t3: T3Cond, gen: dict)`. `generate` calls it when
`audio_prompt_path` is given and otherwise asserts a stored voice. Evidence: `tts.py:161-163,182-206,219-222`;
`mtl_tts.py:221-223,253-278,300-303`; `tts_turbo.py:183-186,241-270,285-288`; VC analogue at `vc.py:48-51,76-81,88-91`.
Reason: `s3gen/s3gen.py:177-180` marks a precomputed `ref_dict` as the "prod API". Within it, the clip is loaded at
`S3GEN_SR` (24 kHz), resampled to `S3_SR` (16 kHz), then cut to `DEC_COND_LEN` for S3Gen and `ENC_COND_LEN` for T3
(`tts.py:184-194`). Exceptions: VC (§0.1).

**D8. Special text tokens are framed outside T3, and T3 only asserts them.** Evidence: docstring `t3.py:41-43`;
`_ensure_BOT_EOT` `t3.py:33-36`, called at `t3.py:142,254`; framing at `tts.py:240-243`, `mtl_tts.py:319-322`.
Reason: that docstring. Exceptions: the Turbo path uses the HF tokenizer with no SOT/EOT padding
(`tts_turbo.py:295-296`), and `inference_turbo` (`t3.py:393-468`) never calls the assert.

**D9. CFG needs two text rows, built by the pipeline; `T3.inference` always runs two.** Evidence: `mtl_tts.py:317`
(comment "Need two seqs for CFG"); `t3.py:310,375` (the BOS and next-token embeddings are duplicated
unconditionally). **Broken once in-repo**: `tts.py:237-238` doubles only when `cfg_weight > 0` (B1). Exceptions:
`inference_turbo` has no CFG (`t3.py:412`, `cfg_weight=0.0`).

**D10. Speech tokens given to S3Gen must be `< SPEECH_VOCAB_SIZE` (6561).** Evidence: filters at `tts.py:262` and
`tts_turbo.py:308`; the guard that logs when violated at `s3gen/flow.py:164-165`; the embedding size `flow.py:49,76`;
the constant `s3tokenizer.py:19`. Reason: `flow.py:165` message ("out-of-range special tokens found in flow"); the link
to `c1af862` "Potentially fix CUDA error" (blame of `tts.py:262`) is inferred. **Broken once in-repo**:
`mtl_tts.py:339` has no filter (B6). Both filters write the literal `6561`.

**D11. T3 backbone family is chosen by a name looked up in a registry, and the family decides the inference path.**
`LLAMA_CONFIGS[hp.llama_config_name]["model_type"]` sets `is_gpt` (`t3.py:55-56`); registry at
`llama_configs.py:105-109`; `T3Config.n_channels` reads the same registry (`t3_config.py:25-27`); Turbo/Nano pick the
name at `tts_turbo.py:154`. Reason: comment `tts_turbo.py:151-152` ("Turbo/Nano share the same architecture; Nano just
uses a smaller GPT2 backbone and its own T3 checkpoint"). GPT2 models need `input_pos_emb=None`,
`use_perceiver_resampler=False`, `emotion_adv=False`, `speech_cond_prompt_len=375` set together
(`tts_turbo.py:156-159`) because `inference` uses `speech_pos_emb` (`t3.py:307,372`) and `prepare_conditioning` branches
on `is_gpt` (`t3.py:98`) (inferred coupling). Exceptions: none.

**D12. T3 checkpoint state may be wrapped as `{"model": [state]}`, and every loader unwraps it.** Evidence:
`tts.py:146-147`; `mtl_tts.py:206-207`; `tts_turbo.py:164-165`. Reason not recorded (inferred: training-checkpoint
format). Exceptions: none.

**D13. Language-specific text handling is a branch in `MTLTokenizer.encode`, ahead of the `[xx]` language-token
prepend; heavy language dependencies are imported lazily and degrade to a warning.** Evidence: branches and prepend
`tokenizer.py:288-302`; lazy helpers with module-level sentinels `tokenizer.py:59-62,77-82,117-133,237-253`.
Reason: commit subjects `c61ec83` "Make russian-text-stresser optional" and `c375f02` "remove russian-stresser from
dependency". Exceptions: the Cangjie mapping failure is also only a warning (B3); `pykakasi` and `spacy-pkuseg` are
hard dependencies (`pyproject.toml:27-28`) while `dicta_onnx` and `russian_text_stresser` are not listed, so Hebrew
and Russian handling is silently skipped unless installed (see Unknowns).

### Hidden couplings (all inferred; no comment or assert enforces them)

- `ENC_COND_LEN / S3_TOKEN_RATE == speech_cond_prompt_len`: 6 s ↔ 150 (`tts.py:107`, `t3_config.py:18`) and 15 s ↔ 375
  (`tts_turbo.py:112,157`), rate 25 (`s3tokenizer.py:18`). `vc.py:17` defines `ENC_COND_LEN` and never uses it.
- `speaker_embed_size = 256` is set in `t3_config.py:21` and again in `voice_encoder/config.py:4`, with no import
  between them.
- `6561` appears as a literal in `tts.py:262`, `tts_turbo.py:308`, `flow.py:49` and as `start_speech_token` in
  `t3_config.py:11`, besides `SPEECH_VOCAB_SIZE` (`s3tokenizer.py:19`, `s3tokenizer/__init__.py:11-12`).
- `text_tokens_dict_size` must equal the checkpoint's embedding rows: 704, 2454 (`t3_config.py:5,36,41`), 50276
  (`tts_turbo.py:153`, `llama_configs.py:67,102`, and the length check `tts_turbo.py:180`). `T3Config.is_multilingual`
  (`t3_config.py:30`) has no callers since `3f35dfc`.
- Edits to one pipeline's `Conditionals`, `punc_norm` or device block do not reach the others (D1).

## 3. Drift

"Newest" is the commit that last changed the differing line (`git blame -L`), with the date. Class **drift** = no
reason visible; **decision** = reason visible (listed here only because the pipelines differ).

| # | What differs | Sites | Newest variant | Class |
|---|---|---|---|---|
| R1 | Where T3 hyperparameters live: `T3()` default (`tts.py:144`, `t3.py:49-51`), `T3Config.multilingual()` (`mtl_tts.py:204`), or attributes mutated inline (`tts_turbo.py:153-159`) | those | inline in Turbo, `bc58894` 2025-12-15; `nano` branch `5de7a54` | drift |
| R2 | Checkpoint format: `ve.safetensors`/`s3gen.safetensors` (`tts.py:140,153`, `tts_turbo.py:147,171`, `vc.py:55`) vs `ve.pt`/`s3gen.pt` (`mtl_tts.py:200,213`) | those | MTL (`a9742ee`, 2025-09-04) is `.pt`; Turbo (2025-12-15) is safetensors. **Date does not decide.** The HF repo contents probably do (unverified offline) | drift |
| R3 | S3Gen `load_state_dict` strictness: `strict=False` (`tts.py:153`, `vc.py:55`), default (`mtl_tts.py:212`), `strict=True` (`tts_turbo.py:172-174`). `ignore_state_dict_missing` (`s3gen.py:239`, `s3tokenizer.py:29`) is declared and read nowhere | those | Turbo `strict=True`, `bc58894` | drift |
| R4 | Download: per-file `hf_hub_download` with no token (`tts.py:177-180`, `vc.py:71-74`); `snapshot_download` with `revision="main"` and `HF_TOKEN` (`mtl_tts.py:242-250`); `snapshot_download` with token and Xet fallback (`tts_turbo.py:201-224`) | those | Turbo, `5de7a54` 2026-07-21 | drift |
| R5 | CFG row doubling conditional (`tts.py:237`) vs unconditional (`mtl_tts.py:317`) | those | MTL, `a9742ee` 2025-09-04 (`tts.py:237` is `3a013dc` 2025-05-30) | drift (B1) |
| R6 | OOV speech-token filter: present (`tts.py:262`, `tts_turbo.py:308`), absent (`mtl_tts.py:339`) | those | MTL, `a9742ee`, **is the one missing the filter**: counterexample to "newest wins" | drift (B6) |
| R7 | `t3_cond_prompt_tokens = None` before the `if`: only `mtl_tts.py:263` (`tts.py:191-195`, `tts_turbo.py:255-259` lack it; with `speech_cond_prompt_len == 0` they would hit an unbound name, inferred) | those | MTL, `a9742ee` | drift (T10) |
| R8 | "exaggeration changed?" test: tensor compare (`tts.py:225`) vs `float(...)` (`mtl_tts.py:306`); Turbo has none | those | MTL | drift |
| R9 | `generate` parameter order differs in all three: `tts.py:208-218` (sampling args first), `mtl_tts.py:280-291` (`language_id` second), `tts_turbo.py:272-284` | those | Turbo `bc58894` 2025-12-15 / MTL `a9742ee` | drift. Call by keyword |
| R10 | Sampling defaults: Turbo `min_p=0.0, top_p=0.95, exaggeration=0.0, cfg_weight=0.0` and a warning that CFG, min_p and exaggeration are ignored (`tts_turbo.py:272-284,290-291`); MTL repetition penalty 2.0 → 1.2 (`3f35dfc`, message "lower rep_penalty default to 1.2") | those | Turbo/MTL | decision. Related drift: `T3.inference` defaults `top_p=0.95` (`t3.py:242`) while pipelines pass 1.0 (`mtl_tts.py:290`) |
| R11 | End of audio: MTL drops the last token's audio (`mtl_tts.py:348-352`, comment gives the reason); Turbo appends 3 `S3GEN_SIL` tokens (`tts_turbo.py:310-311`, no comment); English does neither | those | MTL `3f35dfc` 2026-05-01 | MTL: decision; Turbo: drift (reason not recorded) |
| R12 | Turbo-only reference handling: `>5 s` assert (`tts_turbo.py:245`) and loudness norm (`tts_turbo.py:228-239,247-248`) | those | Turbo `bc58894` | drift (plausibly model-specific, inferred) |
| R13 | Max speech length: literal `1000  # TODO: use the value in config` (`tts.py:249`, `mtl_tts.py:328`), `max_gen_len=1000` (`t3.py:394`), while `T3Config.max_speech_tokens = 4096` (`t3_config.py:14`) is read only for the position table (`t3.py:80`) | those | n/a | drift (B7) |
| R14 | `punc_norm`: Turbo lacks the `"..."` and `";"` rules (`tts.py:39-41,44` vs `tts_turbo.py:46-56`); MTL adds CJK sentence enders (`mtl_tts.py:106`). T10 says "on purpose"; no code or commit says so | those | Turbo `bc58894`; MTL `a9742ee` | drift (reason not recorded) |
| R15 | Messages by `print` (`tts.py:172`, `tts_turbo.py:181,237`) vs `logger` (`tts_turbo.py:291`, `t3.py:6`) | those | Turbo mixes both | drift |
| R16 | Public export: MTL yes (`__init__.py:11`), Turbo no | those | Turbo `bc58894` omitted it | drift (CLAUDE.md states it, no reason) |
| R17 | Wrong annotations: `tts_turbo.py:120` (see §0.5); `mtl_tts.py:230` types `device` as `torch.device` but compares it to `"mps"` (`:234`) | those | n/a | drift |
| R18 | T3 file selection: MTL has a resolver and registry (`mtl_tts.py:22-28,58-68`); English and Turbo hard-code the filename (`tts.py:145`, `tts_turbo.py:162`) | those | MTL `3f35dfc` 2026-05-01 | drift |
| R19 | Default checkpoint: README recommends `t3_model="v3"` (`README.md:126-127`); code and demo default to v2 (`mtl_tts.py:22`, `multilingual_app.py:9`, `example_tts.py:24`). `65b1843` changed README only | those | README `65b1843` 2026-06-10 | drift (docs vs code) |

**Drift count: 19 rows = 17 drift, 1 decision (R10, with a minor `top_p` drift noted in the row), 1 split (R11: MTL
decision, Turbo drift).** Four are already filed: R5 (B1), R6 (B6), R7 (T10), R13 (B7).
**Hypothesis "newest wins"** (unverified): where the newest variant is also the fixed one, it holds. B1's own fix is
"like `mtl_tts.py:317`" (R5). It fails for R6 (the newest, MTL, is the one missing the filter) and for R2 (format follows
the HF repo, not the date), and R12/R14 look model-specific. Check each row; do not apply it blindly.

## 4. Where new code goes

No test suite exists; "verify" means a root example script or the CPU smoke test in CLAUDE.md.

### (a) A new language in the multilingual model

Precedent: `a9742ee` (initial MTL: `mtl_tts.py`, `tokenizers/tokenizer.py`, `t3_config.py`, `__init__.py`,
`multilingual_app.py`, `example_tts.py`, `pyproject.toml`, README) and `bf169fe` (v2: vocab 2352 → 2454 in
`t3_config.py:30,41`, a Russian branch in `tokenizer.py:297-298`, new checkpoint filenames in `mtl_tts.py`).
**No commit adds one language to an existing checkpoint.** `65b1843` (single language pack) is README-only.
Files, in order:

1. Outside the repo: the `[xx]` token must be in the vocab JSON (`tokenizer.py:300-302` prepends it) and the T3 weights
   must be trained for it. Cached v2 vocab has `[sk] [cs] [hu] [ro] [bg] [vi] [ea]` beyond the 23 (Unknowns).
2. `t3_config.py:30-31,38-41`, only if the vocab size changes (`bf169fe`); it must match the checkpoint.
3. `mtl_tts.py:31-55` `SUPPORTED_LANGUAGES`; it gates `generate` (`mtl_tts.py:292-298`), is exported
   (`__init__.py:11`) and drives `multilingual_app.py:123,132`.
4. `tokenizer.py:288-298`: an `elif` branch plus a lazy helper in the style of `tokenizer.py:77-82` (D13); then
   `pyproject.toml` if a package is needed (pins out of scope).
5. `mtl_tts.py:106` `sentence_enders` if the script has its own full stops (inferred from the CJK precedent).
6. `multilingual_app.py:16` `LANGUAGE_CONFIG`, README language list (`README.md:144`).

Single Language Pack models live in separate HF repos (`README.md:149-160`); `REPO_ID` is fixed (`mtl_tts.py:21`),
so `from_pretrained` cannot fetch them. `from_local(dir, device, t3_model=...)` is the only in-code path (inferred).

### (b) A new T3 checkpoint variant (same architecture)

Precedent: `3f35dfc` (v3). Files in order: `mtl_tts.py:23-28` add the registry entry (any `*.safetensors` name already
passes `mtl_tts.py:63-64`, but `from_pretrained` downloads only the resolved name from `REPO_ID`,
`mtl_tts.py:21,247`); then `multilingual_app.py:9`, the `example_tts.py` comment, README. Do not move
`DEFAULT_MULTILINGUAL_T3_MODEL` (`mtl_tts.py:22`) without the maintainers (R19). Caution: `3f35dfc` also changed
behaviour in code that is not conditional on the variant (`mtl_tts.py:348-352`, the default penalty), so v2 changed
with v3. For English or Turbo there is **no selection mechanism**; the closest sites are `tts.py:145` and
`tts_turbo.py:162`. A variant with a different architecture is case (c).

### (c) A new model size, like Nano

Precedent: `5de7a54`. Files in order: `t3/llama_configs.py` (new dict, register at `:105-109`; keep `hidden_size` and
`n_embd` equal, `t3_config.py:27` reads `hidden_size`); `tts_turbo.py` (repo constant `:26-27`, backbone name `:154`,
checkpoint name `:162`, repo choice `:202`, `model_label` `:132,189`); a root example (`example_tts_nano.py`); README.
The vocab sizes (50276, 6563) and the S3Gen meanflow weights (`tts_turbo.py:153-155,170-171`) are shared. The
flag is a bool (`tts_turbo.py:136,192`); a third size needs a mapping instead. **No precedent** for that, or for a
non-GPT2 size (only `Llama_520M` exists); closest sites are the above and `t3.py:55-63`.

### (d) A new paralinguistic tag or tokenizer rule

- **Turbo/Nano tag:** no tag list exists in `src/`; the tokens are inferred to live in the checkpoint's tokenizer files
  (`AutoTokenizer.from_pretrained(ckpt_dir)`, `tts_turbo.py:177`; the Turbo snapshot is not cached, so unverified).
  Repo-side files: `gradio_tts_turbo_app.py:9-12` `EVENT_TAGS`, README (`README.md:30,44`); `punc_norm`
  (`tts_turbo.py:30-66`) must not alter bracketed text. A new tag needs a retrained model; **no precedent** (`bc58894`
  introduced the tags with the model).
- **MTL rule:** `tokenizer.py` `encode` (`:285-305`), precedent `bf169fe` (Russian). The `lowercase` and
  `nfkd_normalize` flags exist (`tokenizer.py:268-285`) but `generate` never passes them (`mtl_tts.py:316`).
- **English rule:** `EnTokenizer.encode` only maps spaces (`tokenizer.py:35-42`); text rules go in `punc_norm`
  (`tts.py:22-61`). No precedent for either.

### (e) A new conditioning input in `Conditionals`

**No precedent commit.** `cond_enc.py` was last touched in the initial commit `f42d31a`. Closest sites: `emotion_adv`
(`t3_config.py:23`, `cond_enc.py:54-57,84-88`) and the unimplemented `clap_emb` (`cond_enc.py:19,73-75`). Files in order:

1. `t3_config.py`: a flag defaulting to off, since Turbo overrides defaults the same way (`tts_turbo.py:159`).
2. `cond_enc.py:12-22`: an **optional field with a default**, because shipped `conds.pt` files load with
   `T3Cond(**kwargs['t3'])` (`tts.py:103`) and will lack it. `T3Cond.to` (`cond_enc.py:24-30`) calls
   `v.view(-1)[0]`, so a tensor field must be non-empty (inferred).
3. `cond_enc.py:54-62,91-96`: the module and its slot in the concat. **The order is fixed by trained weights.**
4. `t3.py:92-100` if the input needs embedding.
5. `prepare_conditionals` in all three: `tts.py:182-206`, `mtl_tts.py:253-278`, `tts_turbo.py:241-270`.
6. **The exaggeration rebuild** (`tts.py:227-231`, `mtl_tts.py:308-312`) re-creates `T3Cond` from three named fields.
   A new field is silently dropped on the next `generate` call that changes `exaggeration`.
7. If S3Gen-side: `embed_ref` (`s3gen/s3gen.py:165-171`) → `flow.inference(**ref_dict)` (`s3gen.py:227`) needs a
   matching argument; `vc.py:76-81` also calls `embed_ref`.
8. Retrained weights.

### (f) A new entry-point class

Precedents: `a9742ee` (MTL: new module, `__init__.py` export, example, gradio app, README, `pyproject.toml`) and `bc58894`
(Turbo: new module, example, gradio app, README, plus model changes in `t3.py` `inference_turbo`, `llama_configs.py` and
`s3gen/{flow,flow_matching,decoder,s3gen}.py`, so vendored S3Gen code is a real seam). Files in order:

1. Model-level changes first (`t3_config.py`, `llama_configs.py`, `t3.py`, `s3gen/*`), if the architecture differs.
2. `src/chatterbox/<name>.py` honouring D1, D2, D4-D12: own `REPO_ID`, `Conditionals`, `punc_norm`, CPU-first logic,
   MPS fallback, `weights_only=True`, token filter, watermark.
3. `__init__.py:9-11` only if the maintainers want it public (MTL yes, Turbo no; Unknowns).
4. A root `example_*.py`; a `gradio_*_app.py` (copy the module-level load of `gradio_vc_app.py`, not the per-session
   `gr.State` load, B5); README.
5. `pyproject.toml` only for new dependencies.

## 5. Unknowns (maintainers only)

1. Are the extra language tokens `[sk] [cs] [hu] [ro] [bg] [vi] [ea] [ipa]` in the **v2** vocab
   (`~/.cache/huggingface/hub/models--ResembleAI--chatterbox/snapshots/*/grapheme_mtl_merged_expanded_v1.json`, 2454
   entries) trained, or leftovers? This decides how cheap (a) is.
2. The same vocab holds `[UH]`, `[UM]` and event tags (`[sigh]`, `[gasp]`, ...), but `tokenizer.py:273-274` lowercases input,
   so uppercase tokens look unreachable (inferred). Usable or not?
3. Why does MTL use `.pt` for `ve` and `s3gen` while the others use safetensors (R2)? Does the HF repo contain both?
4. Why `strict=False` for English and VC S3Gen (R3), and why does `ignore_state_dict_missing` exist without a reader?
5. Is Turbo's absence from `__init__.py` intended (R16)? Which symbols do outside consumers (ebook2audiobook) import? T10's
   `from chatterbox.mtl_tts import Conditionals` is an assumption.
6. Are the three `punc_norm` differences intended (R14)? T10 says yes; no code says so.
7. Is the Turbo silence suffix (R11), the `>5 s` assert and the loudness norm (R12) tied to the distilled decoder?
8. `matcha/{decoder,flow_matching,transformer}.py` carry no header and Matcha-TTS is not in `README.md:217-223`. What is
   their upstream and licence?
9. Do Single Language Pack checkpoints load through `from_local(..., t3_model=...)`, and with which vocab and `T3Config`?
10. Are Turbo tags defined only in the checkpoint tokenizer files? (Turbo snapshot not cached.)
11. When does the multilingual default move from v2 to v3 (R19)?
12. Are Hebrew and Russian preprocessing meant to be optional installs (D13)? README never says to install them.
13. Is the `speech_cond_prompt_len` ↔ `ENC_COND_LEN` link intended, and may `vc.py:17` and `T3Config.is_multilingual` go?

## Unread

Not opened in full (read only by header, imports, class list or grep): `s3gen/{hifigan,decoder,xvector,f0_predictor}.py`,
`s3gen/matcha/*`, all of `s3gen/transformer/*`, `s3gen/utils/{mask,mel,class_utils}.py`, `voice_encoder/melspec.py`,
`t3/modules/perceiver.py`, `t3/inference/t3_hf_backend.py`, and every root script. Read in part only: `s3gen/flow.py`
(`:49-110`, `:150-198`), `s3gen/flow_matching.py` (grep), `voice_encoder/voice_encoder.py` (head and defs),
`s3tokenizer/s3tokenizer.py` (first 60 lines). No decision above depends on their internals.
