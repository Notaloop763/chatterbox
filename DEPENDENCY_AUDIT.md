# Dependency API-surface audit

Branch `relax-dependency-pins`, audited 2026-10-18 against `57c2c70`. Read-only: `pyproject.toml` and every other file are unchanged.
Companion to `DEPENDENCY_MIGRATION.md` (whose targets are hypotheses), `BUGS.md` (**B#**) and `TECH_DEBT.md` (**T#**).

Evidence tags: **read** = I read the source (installed wheel or repo); **inferred** = derived, not seen directly.
Verdicts (per handoff): **holds** = symbol found at both versions; **breaks** = missing or changed (quoted); **unknown** = reason given.

## 0. Method and what the evidence is

- Python 3.12.15 only. Three wheel-only venvs from `uv pip compile`, torch/torchaudio from the CPU index (`+cpu` builds):
  **P** = the exact `pyproject.toml` pins (numpy 1.26.4, torch 2.6.0, transformers 5.2.0, diffusers 0.29.0, safetensors 0.5.3, gradio 6.8.0, librosa 0.11.0, pykakasi 2.3.0, conformer 0.3.2);
  **T** = the migration "target floor": every `>=` at its floor, torch/torchaudio 2.7.1, numpy uncapped (2.5.4);
  **N** = nothing pinned, newest the resolver gives (torch 2.14.1, torchaudio 2.11.0, transformers 5.20.1, diffusers 0.41.0, librosa 1.0.0, safetensors 0.8.0, gradio 6.30.0, numpy 2.5.4).
- **Caveat: none of the venvs has the real perth pin.** `pyproject.toml` pins `resemble-perth @ git+...@master`; P, T and N all got PyPI `resemble-perth` 1.0.1. Git `master` (1.1.0, commit `ff1c8ac`) was cloned as data and imported from its source tree in P and N only. So "P" is not the pinned configuration for perth (see R1).
- Checks run: (a) every module under `src/chatterbox` imported in each venv, plus the public `chatterbox`, `chatterbox.tts_turbo`, `chatterbox.vc`, `chatterbox.mtl_tts` imports, plus the import statements of the 9 root scripts (never the scripts themselves); (b) every inventory symbol resolved by `getattr` chain; (c) every call site's positional/keyword shape bound against `inspect.signature` (nothing called); (d) source read for the rest. No library function was called: no constructor, no forward pass. Python ran with `-I`; `HF_HUB_OFFLINE=1`.
- Result of (a): **42/42 modules and all 4 public imports succeed in P, T and N** (0 failures, 0 captured warnings at import). Result of (c): 0 bind failures; 289 calls have no inspectable signature (C builtins, almost all `torch.*`), 6 use `*`/`**` (read by hand below). **There is no import-time failure on Python 3.12 at any of the three versions.** Everything ranked in §3 fails later: at construction, at a call, or in the future.
- The "resolves" column in §7 means the attribute exists **and is not `None`** (re-run over all 242 symbols in P, T and N). Exceptions: `perth.PerthImplicitWatermarker` (`None`, R1), `numpy.newaxis` (`None` by definition), and the two optional packages (not installed).
- Inventory: 1,322 rows from an AST scan (imports plus attribute chains) of 59 files (50 under `src/` incl. `__init__.py`, + 9 root scripts; 46 of them import something third-party), 23 third-party packages. Mechanical, not sampled. Raw TSVs: scratchpad `.../scratchpad/tools/inv-classified.tsv` (one row per `file:line`).

## 1. Summary per pin

"Newest" = newest on PyPI for Python 3.12 (`pip index versions`, 2026-10-18). "N gets" = what an unbounded install resolves, which can differ.

| Package (pin today) | Migration target | Newest (N gets) | Verdict | Worst finding | Evidence |
|---|---|---|---|---|---|
| `resemble-perth` (git `@master`) | "PyPI release if one exists" | PyPI 1.0.1 (git master is 1.1.0) | **breaks** for PyPI 1.0.1; **holds** for master | R1: PyPI 1.0.1 imports `pkg_resources`; with setuptools >=82 the class is `None` and every pipeline raises `TypeError` in `__init__` | **read** (§3 R1: wheels + master clone; import-tested) |
| `diffusers==0.29.0` | `>=0.29` | 0.41.0 | **holds** now | R2: `LoRACompatibleLinear.__init__` calls `deprecate(..., "1.0.0", ...)`, which raises `ValueError` once diffusers >=1.0.0; FutureWarning today | **read** (§3 R2: `lora.py`, `deprecation_utils.py`, diffed attention code) |
| `librosa==0.11.0` | `>=0.11` | 1.0.0 (needs Python >=3.12 and numpy >=2.1) | **holds** (one path **breaks**) | R3: audioread fallback removed; reference clips libsndfile cannot decode now raise | **read** (§3 R3: `librosa/core/audio.py` 0.11 vs 1.0, METADATA) |
| `torchaudio==2.6.0` | `>=2.6` | 2.11.0 (N: with torch 2.14.1) | **holds** in the library; **breaks** `example_*.py` | R4: `torchaudio.save` needs torchcodec from 2.9; torchcodec is in no lock | **read** (§3 R4: torchaudio 2.11/2.7.1 `__init__.py`) |
| `torch==2.6.0` | `>=2.6`, try 2.7.1 | 2.14.1 | **holds** | R5: `torch.backends.cuda.sdp_kernel` FutureWarning on every generate (still present in 2.14.1) | **read** (§3 R5: torch 2.14.1 source); import-tested |
| `transformers==5.2.0` | `>=5.2` | 5.20.1 | **holds** (read; nothing executed) | `LlamaConfig`/`GPT2Config` became `@strict` dataclasses; legacy keys still accepted (§4.1). Runtime unrun | **read** (§4.1); **inferred** for runtime behaviour |
| `numpy<2` (py<3.13) | drop the cap | 2.5.4 (2.2.6 on py3.10) | **holds** at import and bind | numeric drift (NEP 50) not checked; librosa 1.0 forces numpy >=2.1 | import + bind **read**; numerics **inferred** |
| `safetensors==0.5.3` | `>=0.5.3` | 0.8.0 | **holds** | the exact pin would hold transformers back: transformers 5.20.1 requires `safetensors>=0.8.0` (read) | **read** (transformers 5.20.1 METADATA) |
| `gradio==6.8.0` | `>=6.8` or extra | 6.30.0 | **holds** | none; zero imports in `src/`. Demo-only | **read** (call shapes bound in 6.8.0 and 6.30.0) |
| `conformer==0.3.2` | `>=0.3.2` | 0.3.2 | **dead** | only the dead `ConformerWrapper` uses it; imported at module top (T15) | **read** (§5; T15) |
| `pykakasi==2.3.0` | `>=2.3` | 2.3.0 | **holds** | none (pin = newest) | **read** (identical version in P/T/N) |
| `s3tokenizer` (unpinned) | unchanged | 0.3.0 | **holds** | none; P, T, N all resolve 0.3.0 | **read** (§4.2) |
| `spacy-pkuseg` (unpinned) | unchanged | 1.0.1 | **holds** | none at import; compiled ext imports under numpy 2.5.4 | import **read**; download behaviour **inferred** |
| `pyloudnorm` (unpinned) | unchanged | 0.2.0 | **holds** | none | **read** (§4.2) |
| `omegaconf` (unpinned) | unchanged | 2.4.0 | **dead** | `DictConfig(...)` default argument whose value is never read (T15) | **read** (§5; T15) |
| *imported, not declared:* `scipy`, `tqdm`, `einops`, `huggingface_hub`, `tokenizers` | none | scipy 1.18.1, tqdm 4.70.1, einops 0.8.2, hub 2.2.0 (N gets 1.33.0), tokenizers 0.23.3 | **holds** | arrive transitively today; `einops` is lost if `conformer` is dropped (T15). Fix: declare them | **read** (imports resolved; versions from locks) |
| *optional, guarded:* `dicta_onnx`, `russian_text_stresser`, `importlib_metadata` | none | dicta-onnx 1.0.9; stresser not on PyPI; importlib_metadata dead branch | **unknown** (two) | Hebrew/Russian preprocessing silently skipped unless installed | **read** (guards at `tokenizer.py:128,248`); install state not tested |

## 2. Reach: three numbers per pin

| Package | Pinned | Target (migration doc) | Newest on PyPI | P / T / N resolved |
|---|---|---|---|---|
| numpy | `>=1.24,<2` (py<3.13); `>=2` (py>=3.13) | drop `<2` | 2.5.4 | 1.26.4 / 2.5.4 / 2.5.4 |
| librosa | 0.11.0 | `>=0.11` | 1.0.0 | 0.11.0 / 0.11.0 / 1.0.0 |
| s3tokenizer | none | unchanged | 0.3.0 | 0.3.0 ×3 |
| torch | 2.6.0 (py<3.14) | `>=2.6`; 2.7.1 first | 2.14.1 | 2.6.0 / 2.7.1 / 2.14.1 |
| torchaudio | 2.6.0 (py<3.14) | `>=2.6` | 2.11.0 | 2.6.0 / 2.7.1 / 2.11.0 |
| transformers | 5.2.0 | `>=5.2` | 5.20.1 | 5.2.0 / 5.2.0 / 5.20.1 |
| diffusers | 0.29.0 | `>=0.29` | 0.41.0 | 0.29.0 / 0.29.0 / 0.41.0 |
| resemble-perth | git `@master` | PyPI if one exists | 1.0.1 (master = 1.1.0, `ff1c8ac`) | 1.0.1 ×3 (**not** master) |
| conformer | 0.3.2 | `>=0.3.2` | 0.3.2 | 0.3.2 ×3 |
| safetensors | 0.5.3 | `>=0.5.3` | 0.8.0 | 0.5.3 / 0.5.3 / 0.8.0 |
| spacy-pkuseg | none | unchanged | 1.0.1 | 1.0.1 ×3 |
| pykakasi | 2.3.0 | `>=2.3` | 2.3.0 | 2.3.0 ×3 |
| gradio | 6.8.0 | `>=6.8` / extra | 6.30.0 | 6.8.0 / 6.8.0 / 6.30.0 |
| pyloudnorm | none | unchanged | 0.2.0 | 0.2.0 ×3 |
| omegaconf | none | unchanged | 2.4.0 | 2.4.0 ×3 |

Notes on reach:
- Python 3.10 (`uv pip compile --python 3.10`, **read** from the lock, not installed): numpy 2.2.6, librosa 0.11.0 (1.0.0 and numpy 2.5.4 declare `Requires-Python >=3.12`, **read**), torch 2.14.1, torchaudio 2.11.0, transformers 5.20.1, diffusers 0.41.0. CI and e2a both include 3.10.
- torch and torchaudio no longer move together: torchaudio's newest is 2.11.0 while torch is 2.14.1. N imports fine with that pair (no failure at import, **read** via the check run); runtime is unchecked.
- huggingface_hub: N resolved 1.33.0, but 2.2.0 resolves too when requested (`uv pip compile`, **read**); why the unpinned resolve prefers 1.33.0 was not found (unknown). 2.2.0 was only unpacked as data: `snapshot_download`'s signature is unchanged and `constants.HF_HUB_DISABLE_XET` is still a module constant read dynamically at `file_download.py:2025` (**read**), which is what `tts_turbo.py:221` relies on. Evidence only, never imported.

## 3. Breaks, ranked by blast radius

Tiers: (1) fails at import for every user, (2) fails for every user at construction/load, (3) fails on one path, (4) warning. **Tier 1 is empty**: nothing in P, T or N fails at import.

### R1. Tier 2 (every user, at construction, *not* at import): PyPI `resemble-perth` 1.0.1 under setuptools >=82
- **read**: `perth/__init__.py` (1.0.1) wraps the implicit watermarker import in `try/except ImportError` and sets `PerthImplicitWatermarker = None`. `perth/perth_net/__init__.py:1` is `from pkg_resources import resource_filename`. The wheel declares no `Requires-Dist` at all. `pkg_resources` is in setuptools 80.10.2 and 81.0.0 and absent from 82.0.0 and 82.0.1 (wheels unpacked and listed). All three venvs got setuptools 84.0.0 "via torch" (py3.12 lock), so `import perth` succeeds but `perth.PerthImplicitWatermarker` is `None` in P, T and N.
- Why it is not tier 1: `import chatterbox` succeeds. The failure is `perth.PerthImplicitWatermarker()` in `__init__` of all four pipelines (`tts.py:126`, `mtl_tts.py:175`, `tts_turbo.py:133`, `vc.py:29`; confirmed inside `def __init__`), so `from_pretrained`/`from_local` raises `TypeError: 'NoneType' object is not callable`. The `try/except` hides the real cause. My harness's `hasattr` check passed; only `inspect.signature` exposed it.
- **read**: git `master` (1.1.0, `ff1c8ac`, 2026-09-01) has no `pkg_resources` use and imports cleanly in a venv without it (P and N). So the pinned config works and the migration doc's "PyPI release if one exists" does not: the release exists and is broken.
- A separate agent reported this as "pin `setuptools<80`". Same root cause, but `<80` is stricter than needed: the ceiling is `<82` (80.10.2 and 81.0.0 still ship `pkg_resources`; 82.0.0 does not). Neither is needed if perth stays on git.
- Smallest fix: keep the git dependency and pin it to the commit (`...Perth.git@ff1c8ac55a976971245cdd53c18d6131ca00d993`) until a PyPI release >=1.1.0 exists. Do not switch to PyPI 1.0.1. If forced to, add `setuptools<82` and expect it to conflict with anything else that needs newer setuptools.
- Not run: `PerthImplicitWatermarker()` itself (it loads a model), so master's constructor is unverified beyond import.

### R2. Tier 2, future-dated (every user, at model load): diffusers >=1.0.0
- **read** `diffusers/models/lora.py:~393` (0.41.0; the same line exists in 0.29.0): `LoRACompatibleLinear.__init__` calls `deprecate("LoRACompatibleLinear", "1.0.0", msg)`. `utils/deprecation_utils.py:80-84`: if the installed base version `>= "1.0.0"` it raises `ValueError("The deprecation tuple ... should be removed since diffusers' version ... is >= 1.0.0")`. The file header says it "is deprecated and will be removed soon".
- Live path: `FeedForward` (`matcha/transformer.py:126`, and `SnakeBeta` at `:48`) <- `BasicTransformerBlock` <- `s3gen/decoder.py:148,172,201`. Every `S3Gen()` construction hits it. Newest today is 0.41.0, so this is a time bomb, plus a `FutureWarning` per layer today.
- Smallest fix: replace `LoRACompatibleLinear` with `nn.Linear` at `matcha/transformer.py:13,48,126`. It is an `nn.Linear` subclass whose extra `lora_layer` is `None` and not a parameter, so state-dict keys and outputs are identical (**read**). Fallback: `diffusers<1.0`.
- **Decision (2026-10-23):** take the fallback for now, `diffusers>=0.29,<1.0`; it keeps the
  e2a-resolved 0.41.0 working. Replace `LoRACompatibleLinear` and drop the cap in a follow-up.
- Other diffusers symbols on the live path were compared 0.29 -> 0.41: `Attention.__init__` still accepts every kwarg the repo passes; `AttnProcessor2_0.__call__` and `prepare_attention_mask` differ only by type hints, an unused `norm_q`/`norm_k` (default `None`) and `output_size=` on `repeat_interleave` (**read**, diffed). The `AdaLayerNorm*` branches are not reached (`num_embeds_ada_norm` is never passed).

### R3. Tier 3 (one path): librosa 1.0.0
- **read**: 0.11.0 `librosa/core/audio.py:170-184` falls back to `audioread` when soundfile fails; 1.0.0 has no `audioread` anywhere in that file and its wheel no longer depends on it. `load` is the reference-voice path in all four pipelines (`tts.py:184`, `mtl_tts.py:255`, `tts_turbo.py:243`, `vc.py:78,94`). Clips libsndfile cannot decode (e.g. some `.m4a`/AAC) now raise. Which formats libsndfile covers on a given machine was not tested (inferred).
- Also **read**: 1.0.0 requires `numpy>=2.1` and Python >=3.12, so `librosa>=0.11` silently drags numpy to 2.x on 3.12+, which the migration doc already lists as "likely the main source of fails" for coexistence with e2a.
- Call shapes all bind: `load(path, sr=)`, `resample(y, orig_sr=, target_sr=[, res_type=])`, `filters.mel(sr=, n_fft=, n_mels=[, fmin=, fmax=])` (keyword-only, matches), `stft(..., pad_mode="reflect")` (explicit, so the 0.8 default change does not apply), `effects.trim(top_db=)`.
- Smallest fix: `librosa>=0.11,<1` if reference-clip formats or numpy 1 matter; otherwise accept 1.0 and document the format loss.
- **Decision (2026-10-23):** capped at `librosa>=0.11,<1.0` after reproducing the `.m4a`/AAC
  failure under 1.0 (see `DEPENDENCY_MIGRATION.md` Results). 0.11.0 was retested with numpy 2.5.4.
- Latent, same in both versions (not a pin issue): `voice_encoder.py:262` uses `res_type="kaiser_fast"`, which needs `resampy`, in no requirement list (**read**: `librosa/core/audio.py` lazy-loads it). All callers pass `sample_rate=S3_SR` (16 kHz), so the branch is not taken (inferred from `tts.py:198`, `mtl_tts.py:270`, `tts_turbo.py:262`).

### R4. Tier 3 (one path, examples only): torchaudio >=2.9
- **read** `torchaudio/__init__.py` (2.11.0): `save` "relies on TorchCodec"; `_torchcodec.py:83-84,247-248` re-raise `ImportError` if `torchcodec` is missing. torchcodec is in none of the three locks. 2.7.1's `__init__` has no torchcodec reference (**read**), so the target holds.
- Hit: `torchaudio.save` in `example_tts.py`, `example_tts_turbo.py`, `example_tts_nano.py`, `example_vc.py`, `example_for_mac.py`. The library itself uses only `transforms.Resample` (`s3gen.py:44`) and `compliance.kaldi.fbank` (`xvector.py:50`): both present and signature-compatible in 2.11.0 (**read**).
- Smallest fix: cap `torchaudio<2.9` only if torch <2.9 is wanted anyway; otherwise change the examples to write with `soundfile` (already installed via librosa) or add `torchcodec`.

### R5. Tier 4 (warning on every generate): `torch.backends.cuda.sdp_kernel`
- **read** torch 2.14.1 `backends/cuda/__init__.py:~676-700`: still present, `@deprecated(..., category=FutureWarning)`, "In the future, this context manager will be removed". Used at `perceiver.py:94` on every call (T17). Same family as R2's FutureWarning. Fix: T17 (call `F.scaled_dot_product_attention` directly).

### R6. Not a break, but a hazard for the loosened pins
- Exact `safetensors==0.5.3` is incompatible with transformers 5.20.1 (`Requires-Dist: safetensors>=0.8.0`, **read**); with the pin kept and transformers unbounded, the resolver silently backtracks transformers instead of failing. The migration's `>=0.5.3` removes this.
- `weights_only`: `vc.py:50` omits it (B8). Torch >=2.6 defaults to safe mode, so the target is fine; torch 2.4/2.5 would not be.

## 4. Verdict notes for the calls the binder could not check

### 4.1 transformers (tier-2 if wrong; verdict **holds**, read only)
- `LlamaConfig(**LLAMA_520M_CONFIG_DICT)` (`t3.py:62`), `GPT2Config(**config)` (`t3.py:59`): in 5.20.1 both are `@strict` dataclasses with no `**kwargs` in the signature, wrapped by `wrap_init_to_accept_kwargs` (`configuration_utils.py:139-178`), which passes non-field keys to `__post_init__`, which `setattr`s them (`:496-502`). In 5.2.0 the signature has `**kwargs`. The repo's extra keys (`model_type`, `torch_dtype`, `attn_implementation`, `task_specific_params`, `summary_*`, `n_special`, `n_ctx`, `hidden_size` via `attribute_map`) therefore still land.
- Legacy rope keys: `rope_scaling` + `rope_theta` are converted by `convert_rope_params_to_dict` (5.2.0 `modeling_rope_utils.py:632`, 5.20.1 `:752`) into `rope_parameters` with `rope_theta` set. The llama3 key names the repo passes (`factor`, `low_freq_factor`, `high_freq_factor`, `original_max_position_embeddings`, `rope_type`) are the ones documented for `_compute_llama3_parameters` in 5.20.1. A silent mis-scaling would not raise, so this is **read**, not run.
- `LlamaModel.forward` / `GPT2Model.forward`: both decorated `merge_with_config_defaults` + `capture_outputs`, which pop `return_dict` and honour `output_hidden_states` (`utils/output_capturing.py`), so `t3.py:152` and `t3_hf_backend.py` (`hidden_states[-1]`, `past_key_values`) still work. `cache_position` was dropped from `LlamaModel.forward`; the repo never passes it. `DynamicCache.__len__` exists (`len(past_key_values)` in the backend).
- `T3HuggingfaceBackend(LlamaPreTrainedModel, GenerationMixin)`: `PreTrainedModel.__init__` diff 5.2.0 -> 5.20.1 only adds `kernel_config`, `allow_all_kernels`, `generation_config_class` (read, diffed).
- Logits warpers: `Temperature/TopK/TopP/MinP...Warper`, `RepetitionPenaltyLogitsProcessor`, `LogitsProcessorList` all still in `generation/logits_process.py` (5.20.1:63-720), not aliased or deprecated.
- State-dict drift (the `t3.load_state_dict(t3_state)` path, strict): diffed the attributes `__init__` defines on `LlamaModel`, `LlamaDecoderLayer`, `LlamaAttention`, `LlamaMLP`, `LlamaRMSNorm`, `GPT2Model`, `GPT2Block`, `GPT2MLP`, diffusers `GELU`, `GEGLU`: no names added or removed. `GPT2Attention` gains only `scaling` (a float). `LlamaRotaryEmbedding.inv_freq` is `persistent=False` in both. diffusers `Attention` gains attributes (`pre_only`, `norm_added_q/k`, ...) that create submodules only when `added_kv_proj_dim` is set; the repo never sets it (inferred from the call at `matcha/transformer.py:196`). **read**, not loaded: no checkpoint was available.
- `AutoTokenizer.from_pretrained(ckpt_dir)` (`tts_turbo.py:177`) and the call `self.tokenizer(text, return_tensors="pt", padding=True, truncation=True)`: symbol present; behaviour against the Turbo `tokenizer.json` is **unknown** (not run, no tokenizer files).

### 4.2 Other manual reads
- `huggingface_hub.snapshot_download(**download_kwargs)` (`tts_turbo.py:208,222`): kwargs are not statically visible; `mtl_tts.py:243`, `tts.py:178`, `vc.py:72`, `tokenizer.py:170` bind. The Xet fallback flips `hf_constants.HF_HUB_DISABLE_XET`; still a dynamically read constant in 1.33.0 and 2.2.0 (**read**).
- `s3tokenizer` 0.3.0: `S3Tokenizer(S3TokenizerV2)` calls `super().__init__(name)`; 0.3.0 `__init__(self, name, config=ModelConfig())`. Uses `self.device`, `quantize(mel, mel_len)`, `utils.padding`: all present (**read**). 0.3.0 `quantize` adds automatic long-audio handling (>30 s); reference clips are shorter (inferred).
- `pykakasi.kakasi().convert(text)` -> dicts with `orig`, `hira` (`tokenizer.py:85-95`): version identical in all venvs, not independently re-read. Failures there are swallowed by `except Exception` and degrade Japanese silently.
- `spacy_pkuseg.pkuseg()` has a `download.py` using `requests`/`urlopen` (**read** file exists); that the default constructor downloads a model on first use is **inferred**, not checked.
- `pyloudnorm.Meter(sr).integrated_loudness(wav)` (`tts_turbo.py:230`): signatures match; uses `scipy.signal.lfilter`; no numpy-1-only names (grep). Turbo-only.
- numpy: no removed-in-2 alias (`np.float_`, `np.product`, `np.trapz`, `np.in1d`, ...) appears in the repo (grep, 0 hits); `numpy.lib.stride_tricks.as_strided` still public. Type-promotion (NEP 50) effects on `hifigan.py:354`, `voice_encoder.py` are **inferred** harmless, not measured.
- gradio demos: constructors bind in 6.8.0 and 6.30.0; `Blocks.launch(share, mcp_server)`, `Blocks.queue(max_size, default_concurrency_limit)` bind in both; `Button.click`, `Blocks.load`, `Dropdown.change` expose the same `fn/inputs/outputs/api_name/show_progress/queue` parameters in both (25 vs 24 parameters, the extra one not identified).

## 5. Dead code: what deleting #15/#16 retires

Counted by script over the inventory against the ranges in T16 (line ranges re-checked in the files) plus the import lines whose every use is inside them:
- **9 dead import-statement rows**, 7 statements: `flow.py:24` (omegaconf), `matcha/decoder.py:7` (conformer), `matcha/decoder.py:9` (einops: pack, rearrange, repeat), `hifigan.py:27` (`torch.nn.utils.remove_weight_norm`), `matcha/text_encoder.py:5,6,7` (torch, torch.nn, einops).
- 1,322 rows total: 1,189 live, 133 dead (127 torch, 4 einops, 1 conformer, 1 omegaconf).
- **Retires 2 pins** (`conformer`, `omegaconf`). `einops` stays needed (live in `s3gen/decoder.py` and `perceiver.py`) and must be declared (T15).
- Dead but executed at import today: `flow.py:24` + the `DictConfig(...)` default at `flow.py:57`, `matcha/decoder.py:7,9`, `hifigan.py:27`. They run on the live import chain, so a break in them is tier 1 until deleted. Only `text_encoder.py` is never imported. All five resolve fine in P, T and N.
- `hifigan.py:27` (`remove_weight_norm`) still exists in torch 2.14.1 (`weight_norm.py:149`); deleting it is cleanup, not a fix.

## 6. Not checked

- **Numerics**: no forward pass, no audio compared. "Holds" never means "same output" (config rope handling, NEP 50, librosa 1.0 resampling/mel defaults, diffusers attention).
- **Python 3.10 and 3.11, 3.13, 3.14**: only the 3.10 resolution (lock read) was done; nothing installed or imported. The `torch>=2.9` / `numpy>=2` branches of `pyproject.toml` for 3.13/3.14 untouched.
- **GPU / MPS / ROCm / XPU**: CPU wheels only (`+cpu`); the PyPI CUDA wheels differ in setuptools/nvidia dependency sets. `sdp_kernel` flags only matter on CUDA.
- **torch 2.14.1 with torchaudio 2.11.0 at runtime**: import only. **Mixed combinations** (target torch + newest diffusers, etc.): only P/T/N were built.
- **Perth master**: not installed; imported from its source tree. Its constructor (loads a model) was not run. Reason for P's perth being PyPI: the pin is a git URL; the venvs were wheel-only.
- **Weights/tokenizer files**: absent, so strict `load_state_dict` and Turbo's `AutoTokenizer` are read-only verdicts.
- Why unpinned `huggingface_hub` resolves 1.33.0 when 2.2.0 resolves too; `spacy_pkuseg` download behaviour; the one extra parameter on gradio 6.30 listeners.
- `dicta_onnx` and `russian_text_stresser` (optional): not installed; `russian-text-stresser` is not on PyPI. e2a's `coqui-tts` numpy requirement (migration doc) not checked. Windows/macOS.
- Row-level verdicts in §7 are mechanical (resolves in P/T/N + call shape binds). Where a row says **holds** but the call has `*`/`**` or no signature (`nosig`, `stararg`), the real evidence is §3-§4 or "symbol exists" only (the handoff's definition of **holds**).

## 6b. Names selected by string (registry rows)

The AST inventory cannot see these; each was found by reading the call chain and checked against the registry in P (transformers 5.2.0, diffusers 0.29.0, torch 2.6.0) and N (5.20.1, 0.41.0, 2.14.1) by attribute lookup (nothing called). A missing key would fail at construction for every user and no import test catches it.

| Package | Name | Selected at | Registry | P | N | Verdict |
|---|---|---|---|---|---|---|
| torch | `torch.nn.SiLU` (string `"swish"`) | `upsample_encoder.py:178` `COSYVOICE_ACTIVATION_CLASSES[activation_type]` -> `class_utils.py:43` `getattr(torch.nn, "SiLU", Swish)` | `torch.nn` | ✓ | ✓ | **holds** |
| torch | `Hardtanh, Tanh, ReLU, SELU, GELU, Identity` | `class_utils.py:23-45`, only the `"swish"` entry is selected by `s3gen.py:64` | `torch.nn` | ✓ | ✓ | **holds** (rest unselected) |
| diffusers | `get_activation("silu")` | `matcha/decoder.py:92` via `TimestepEmbedding(act_fn="silu")` (`s3gen/decoder.py:126-130`); `post_act_fn` is `None` | `ACTIVATION_FUNCTIONS` (0.29) -> renamed `ACT2CLS` (0.41), both contain `"silu"` | ✓ | ✓ | **holds**; 0.41 builds a fresh module per call instead of a shared instance (**read**), no state either way. The repo does not import the dict |
| diffusers | `activation_fn="gelu"` | `s3gen/decoder.py:111` -> `FeedForward` (`matcha/transformer.py:109`), a local string compare to `diffusers.GELU(dim, inner_dim)` | local | n/a | n/a | **holds** (symbol rows above) |
| transformers | `hidden_act="silu"` | `llama_configs.py` -> `LlamaMLP` | `ACT2FN` | ✓ | ✓ | **holds** |
| transformers | `activation_function="gelu_new"` | `llama_configs.py` GPT2 configs -> `GPT2MLP` | `ACT2FN` | ✓ | ✓ | **holds** |
| transformers | `rope_type="llama3"` | `llama_configs.py` `rope_scaling` | `ROPE_INIT_FUNCTIONS` | ✓ | ✓ | **holds** (key names checked in §4.1) |
| transformers | `attn_implementation="sdpa"` | `llama_configs.py` | `ALL_ATTENTION_FUNCTIONS` | ✓ | ✓ | **holds** |
| s3gen (local) | `"linear"`, `"rel_pos_espnet"`, `"rel_selfattn"` | `s3gen.py:64-76` -> `COSYVOICE_{SUBSAMPLE,EMB,ATTENTION}_CLASSES` | repo-local classes | n/a | n/a | not third-party; unaffected by pins |

## 7. Inventory and verdicts

One row per (package, symbol). `where` lists `file:line` (paths relative to `src/chatterbox/` or the repo root for scripts); more than 4 sites are collapsed to `file×count`, with exact lines in the TSV above. `P/T/N` = symbol resolves in the pinned / target-floor / newest venv. `call-binds (N)` = outcome of binding every call shape against the signature in N: `binds`, `nosig` (C builtin, no signature), `stararg` (call uses `*`/`**`, read by hand), `unresolved` (resolved to `None` or not installed).

| package | symbol | where (`file:line`) | live/dead | P/T/N resolves | call-binds (N) | verdict |
|---|---|---|---|---|---|---|
| conformer | `conformer.ConformerBlock` | models/s3gen/matcha/decoder.py:7 (dead) | dead | ✓✓✓ | - | n/a (dead); resolves everywhere |
| dicta_onnx | `dicta_onnx.Dicta` | models/tokenizers/tokenizer.py:123 | live | ✗✗✗ | unresolved | **unknown**: optional, not installed; guarded by `except ImportError` (tokenizer.py:128) |
| diffusers | `diffusers.models.activations.get_activation` | models/s3gen/matcha/decoder.py:8 | live | ✓✓✓ | binds | **holds** |
| diffusers | `diffusers.models.attention.AdaLayerNorm` | models/s3gen/matcha/transformer.py:5 | live | ✓✓✓ | binds | **holds** |
| diffusers | `diffusers.models.attention.AdaLayerNormZero` | models/s3gen/matcha/transformer.py:5 | live | ✓✓✓ | binds | **holds** |
| diffusers | `diffusers.models.attention.ApproximateGELU` | models/s3gen/matcha/transformer.py:5 | live | ✓✓✓ | binds | **holds** |
| diffusers | `diffusers.models.attention.GEGLU` | models/s3gen/matcha/transformer.py:5 | live | ✓✓✓ | binds | **holds** |
| diffusers | `diffusers.models.attention.GELU` | models/s3gen/matcha/transformer.py:5 | live | ✓✓✓ | binds | **holds** |
| diffusers | `diffusers.models.attention_processor.Attention` | models/s3gen/matcha/transformer.py:12 | live | ✓✓✓ | binds | **holds** |
| diffusers | `diffusers.models.lora.LoRACompatibleLinear` | models/s3gen/matcha/transformer.py:13 | live | ✓✓✓ | binds | **holds** now; FutureWarning on every construction; **breaks** at diffusers 1.0.0 (§3 R2) |
| diffusers | `diffusers.utils.torch_utils.maybe_allow_in_graph` | models/s3gen/matcha/transformer.py:14 | live | ✓✓✓ | - | **holds** |
| einops | `einops.pack` | models/s3gen/decoder.py:17, models/s3gen/matcha/decoder.py:9 (dead) | live+dead | ✓✓✓ | binds | **holds** |
| einops | `einops.rearrange` | models/s3gen/decoder.py:17, models/s3gen/matcha/decoder.py:9 (dead), models/s3gen/matcha/text_encoder.py:7 (dead), models/t3/modules/perceiver.py:9 | live+dead | ✓✓✓ | binds | **holds** |
| einops | `einops.repeat` | models/s3gen/decoder.py:17, models/s3gen/matcha/decoder.py:9 (dead) | live+dead | ✓✓✓ | binds | **holds** |
| gradio | `gradio` | gradio_tts_app.py:4, gradio_tts_turbo_app.py:4, gradio_vc_app.py:2, multilingual_app.py:6 | live | ✓✓✓ | - | **holds** |
| gradio | `gradio.Accordion` | gradio_tts_app.py:58, gradio_tts_turbo_app.py:154, multilingual_app.py:287 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Audio` | 8 sites: gradio_tts_app.py×2, gradio_tts_turbo_app.py×2, gradio_vc_app.py×2, multilingual_app.py×2 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Blocks` | gradio_tts_app.py:44, gradio_tts_turbo_app.py:114, multilingual_app.py:242 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Button` | gradio_tts_app.py:65, gradio_tts_turbo_app.py:133, gradio_tts_turbo_app.py:149, multilingual_app.py:291 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Checkbox` | gradio_tts_turbo_app.py:161 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Column` | 6 sites: gradio_tts_app.py×2, gradio_tts_turbo_app.py×2, multilingual_app.py×2 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Dropdown` | multilingual_app.py:261 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Interface` | gradio_vc_app.py:17 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Markdown` | gradio_tts_turbo_app.py:115, multilingual_app.py:243, multilingual_app.py:251, multilingual_app.py:275 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Number` | gradio_tts_app.py:59, gradio_tts_turbo_app.py:155, multilingual_app.py:288 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Row` | gradio_tts_app.py:47, gradio_tts_turbo_app.py:119, gradio_tts_turbo_app.py:130, multilingual_app.py:252 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Slider` | 14 sites: gradio_tts_app.py×6, gradio_tts_turbo_app.py×5, multilingual_app.py×3 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.State` | gradio_tts_app.py:45, gradio_tts_turbo_app.py:117 | live | ✓✓✓ | binds | **holds** |
| gradio | `gradio.Textbox` | gradio_tts_app.py:49, gradio_tts_turbo_app.py:121, multilingual_app.py:255 | live | ✓✓✓ | binds | **holds** |
| huggingface_hub | `huggingface_hub.constants` | tts_turbo.py:220 | live | ✓✓✓ | - | **holds** |
| huggingface_hub | `huggingface_hub.constants.HF_HUB_DISABLE_XET` | tts_turbo.py:221 | live | ✓✓✓ | - | **holds** |
| huggingface_hub | `huggingface_hub.hf_hub_download` | models/tokenizers/tokenizer.py:8, tts.py:8, vc.py:6 | live | ✓✓✓ | binds | **holds** |
| huggingface_hub | `huggingface_hub.snapshot_download` | mtl_tts.py:10, tts_turbo.py:12 | live | ✓✓✓ | binds,starargs | **holds** |
| importlib_metadata | `importlib_metadata.version` | __init__.py:4 | live | ✓✓✓ | binds | **holds** (dead branch: `from importlib.metadata` always succeeds on >=3.8; `requires-python >=3.10`) |
| librosa | `librosa` | 7 sites: models/s3tokenizer/s3tokenizer.py×1, models/voice_encoder/melspec.py×1, models/voice_encoder/voice_encoder.py×1, mtl_tts.py×1, tts.py×1, tts_turbo.py×1, vc.py×1 | live | ✓✓✓ | - | **holds** |
| librosa | `librosa.effects.trim` | models/voice_encoder/voice_encoder.py:267 | live | ✓✓✓ | binds | **holds** |
| librosa | `librosa.filters.mel` | models/s3gen/utils/mel.py:3, models/s3tokenizer/s3tokenizer.py:39, models/voice_encoder/melspec.py:11 | live | ✓✓✓ | binds | **holds** |
| librosa | `librosa.load` | 5 sites: mtl_tts.py×1, tts.py×1, tts_turbo.py×1, vc.py×2 | live | ✓✓✓ | binds | **holds** for libsndfile formats; **breaks** for audioread-only formats on 1.0.0 (§3 R3) |
| librosa | `librosa.resample` | models/voice_encoder/voice_encoder.py:262, mtl_tts.py:257, tts.py:186, tts_turbo.py:250 | live | ✓✓✓ | binds | **holds** |
| librosa | `librosa.stft` | models/voice_encoder/melspec.py:57 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy` | 10 sites: gradio_tts_app.py×1, gradio_tts_turbo_app.py×1, multilingual_app.py×1, models/s3gen/hifigan.py×1, models/s3gen/s3gen.py×1, models/s3gen/transformer/embedding.py×1, models/s3gen/utils/mel.py×1, models/s3tokenizer/s3tokenizer.py×1, models/voice_encoder/melspec.py×1, models/voice_encoder/voice_encoder.py×1 | live | ✓✓✓ | - | **holds** |
| numpy | `numpy.abs` | models/voice_encoder/melspec.py:30, models/voice_encoder/melspec.py:36 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.array` | models/voice_encoder/voice_encoder.py:34 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.asarray` | models/voice_encoder/voice_encoder.py:235 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.ceil` | models/s3tokenizer/s3tokenizer.py:66, models/voice_encoder/voice_encoder.py:190 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.clip` | models/voice_encoder/melspec.py:22 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.concatenate` | models/voice_encoder/voice_encoder.py:103, models/voice_encoder/voice_encoder.py:194 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.cumprod` | models/s3gen/hifigan.py:354 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.cumsum` | models/voice_encoder/voice_encoder.py:194 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.dot` | models/voice_encoder/melspec.py:42 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.float32` | models/s3gen/hifigan.py:379, models/voice_encoder/melspec.py:48, models/voice_encoder/voice_encoder.py:108 | live | ✓✓✓ | - | **holds** |
| numpy | `numpy.full` | models/voice_encoder/voice_encoder.py:103 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.lib.stride_tricks.as_strided` | models/voice_encoder/voice_encoder.py:6 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.linalg.norm` | models/voice_encoder/voice_encoder.py:209 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.log` | models/s3gen/transformer/embedding.py:157 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.log10` | models/voice_encoder/melspec.py:68, models/voice_encoder/melspec.py:76 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.maximum` | models/voice_encoder/melspec.py:68 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.mean` | models/voice_encoder/voice_encoder.py:208 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.ndarray` | 12 sites: multilingual_app.py×1, models/s3gen/s3gen.py×2, models/s3gen/utils/mel.py×1, models/s3tokenizer/s3tokenizer.py×2, models/voice_encoder/voice_encoder.py×6 | live | ✓✓✓ | - | **holds** |
| numpy | `numpy.newaxis` | models/s3gen/transformer/embedding.py:160, models/s3gen/transformer/embedding.py:161 | live | ✓✓✓ (value is `None` by definition) | - | **holds** |
| numpy | `numpy.pi` | models/s3gen/hifigan.py:211, models/s3gen/hifigan.py:212, models/s3gen/hifigan.py:212 | live | ✓✓✓ | - | **holds** |
| numpy | `numpy.power` | models/voice_encoder/melspec.py:72 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.prod` | models/s3gen/hifigan.py:324, models/s3gen/hifigan.py:329 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.random.seed` | gradio_tts_app.py:16, gradio_tts_turbo_app.py:74, multilingual_app.py:168 | live | ✓✓✓ | binds | **holds** |
| numpy | `numpy.round` | models/voice_encoder/voice_encoder.py:77, models/voice_encoder/voice_encoder.py:79 | live | ✓✓✓ | binds | **holds** |
| omegaconf | `omegaconf.DictConfig` | models/s3gen/flow.py:24 (dead) | dead | ✓✓✓ | binds | n/a (dead); resolves everywhere |
| perth | `perth` | mtl_tts.py:7, tts.py:6, tts_turbo.py:8, vc.py:5 | live | ✓✓✓ | - | **holds** |
| perth | `perth.PerthImplicitWatermarker` | mtl_tts.py:175, tts.py:126, tts_turbo.py:133, vc.py:29 | live | `None` ×3 | unresolved | **breaks** for PyPI 1.0.1 (attr is `None`, §3 R1); **holds** for git master 1.1.0 |
| pykakasi | `pykakasi` | models/tokenizers/tokenizer.py:81 | live | ✓✓✓ | - | **holds** |
| pykakasi | `pykakasi.kakasi` | models/tokenizers/tokenizer.py:82 | live | ✓✓✓ | nosig | **holds** |
| pyloudnorm | `pyloudnorm` | tts_turbo.py:9 | live | ✓✓✓ | - | **holds** |
| pyloudnorm | `pyloudnorm.Meter` | tts_turbo.py:230 | live | ✓✓✓ | binds | **holds** |
| russian_text_stresser | `russian_text_stresser.text_stresser.RussianTextStresser` | models/tokenizers/tokenizer.py:243 | live | ✗✗✗ | unresolved | **unknown**: optional, not on PyPI; guarded by `except ImportError` (tokenizer.py:248) |
| s3tokenizer | `s3tokenizer.model_v2.ModelConfig` | models/s3tokenizer/s3tokenizer.py:8 | live | ✓✓✓ | binds | **holds** |
| s3tokenizer | `s3tokenizer.model_v2.S3TokenizerV2` | models/s3tokenizer/s3tokenizer.py:8 | live | ✓✓✓ | - | **holds** |
| s3tokenizer | `s3tokenizer.utils.padding` | models/s3tokenizer/s3tokenizer.py:7 | live | ✓✓✓ | binds | **holds** |
| safetensors | `safetensors.torch.load_file` | mtl_tts.py:9, tts.py:9, tts_turbo.py:11, vc.py:7 | live | ✓✓✓ | binds | **holds** |
| scipy | `scipy.signal` | models/voice_encoder/melspec.py:3 | live | ✓✓✓ | - | **holds** |
| scipy | `scipy.signal.get_window` | models/s3gen/hifigan.py:22 | live | ✓✓✓ | binds | **holds** |
| scipy | `scipy.signal.lfilter` | models/voice_encoder/melspec.py:21 | live | ✓✓✓ | binds | **holds** |
| spacy_pkuseg | `spacy_pkuseg.pkuseg` | models/tokenizers/tokenizer.py:193 | live | ✓✓✓ | binds | **holds** |
| tokenizers | `tokenizers.Tokenizer` | models/tokenizers/tokenizer.py:7 | live | ✓✓✓ | - | **holds** |
| tokenizers | `tokenizers.Tokenizer.from_file` | models/tokenizers/tokenizer.py:22, models/tokenizers/tokenizer.py:258 | live | ✓✓✓ | binds | **holds** |
| torch | `torch` | 44 sites: example_for_mac.py×1, example_tts.py×1, example_tts_nano.py×1, example_tts_turbo.py×1, example_vc.py×1, gradio_tts_app.py×1, gradio_tts_turbo_app.py×1, gradio_vc_app.py×1, multilingual_app.py×1, models/s3gen/decoder.py×1, models/s3gen/f0_predictor.py×1, models/s3gen/flow.py×1, models/s3gen/flow_matching.py×1, models/s3gen/hifigan.py×1, models/s3gen/matcha/decoder.py×1, models/s3gen/matcha/flow_matching.py×1, models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/matcha/transformer.py×1, models/s3gen/s3gen.py×1, models/s3gen/transformer/activation.py×1, models/s3gen/transformer/attention.py×1, models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/embedding.py×1, models/s3gen/transformer/encoder_layer.py×1, models/s3gen/transformer/positionwise_feed_forward.py×1, models/s3gen/transformer/subsampling.py×1, models/s3gen/transformer/upsample_encoder.py×1, models/s3gen/utils/class_utils.py×1, models/s3gen/utils/intmeanflow.py×1, models/s3gen/utils/mask.py×1, models/s3gen/utils/mel.py×1, models/s3gen/xvector.py×1, models/s3tokenizer/s3tokenizer.py×1, models/t3/inference/t3_hf_backend.py×1, models/t3/modules/cond_enc.py×1, models/t3/modules/learned_pos_emb.py×1, models/t3/modules/perceiver.py×1, models/t3/t3.py×1, models/tokenizers/tokenizer.py×1, models/voice_encoder/voice_encoder.py×1, mtl_tts.py×1, tts.py×1, tts_turbo.py×1, vc.py×1 | live+dead | ✓✓✓ | - | **holds** |
| torch | `torch.FloatTensor` | 6 sites: models/s3gen/matcha/transformer.py×4, models/s3tokenizer/s3tokenizer.py×1, models/voice_encoder/voice_encoder.py×1 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.IntTensor` | models/tokenizers/tokenizer.py:32, models/tokenizers/tokenizer.py:282 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.LongTensor` | 15 sites: models/s3gen/matcha/transformer.py×2, models/s3gen/s3gen.py×2, models/s3tokenizer/s3tokenizer.py×1, models/t3/t3.py×10 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.Tensor` | 195 sites: models/s3gen/decoder.py×6, models/s3gen/f0_predictor.py×2, models/s3gen/flow.py×1, models/s3gen/hifigan.py×9, models/s3gen/matcha/text_encoder.py×3 (dead), models/s3gen/s3gen.py×6, models/s3gen/transformer/activation.py×2, models/s3gen/transformer/attention.py×30, models/s3gen/transformer/convolution.py×5, models/s3gen/transformer/embedding.py×24, models/s3gen/transformer/encoder_layer.py×10 (dead), models/s3gen/transformer/encoder_layer.py×10, models/s3gen/transformer/positionwise_feed_forward.py×4, models/s3gen/transformer/subsampling.py×44, models/s3gen/transformer/upsample_encoder.py×18, models/s3gen/utils/mask.py×5, models/s3tokenizer/s3tokenizer.py×4, models/t3/inference/t3_hf_backend.py×4, models/t3/modules/cond_enc.py×1, models/t3/modules/learned_pos_emb.py×1, models/t3/t3.py×1, models/tokenizers/tokenizer.py×2, models/voice_encoder/voice_encoder.py×3 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.abs` | models/s3gen/f0_predictor.py:55, models/s3gen/matcha/text_encoder.py:255 (dead), models/t3/modules/perceiver.py:28 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.all` | models/t3/t3.py:450, models/t3/t3.py:459 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.arange` | 19 sites: models/s3gen/matcha/decoder.py×1, models/s3gen/matcha/text_encoder.py×4 (dead), models/s3gen/transformer/embedding.py×7, models/s3gen/utils/mask.py×2, models/t3/modules/learned_pos_emb.py×1, models/t3/modules/perceiver.py×2, models/t3/t3.py×2 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.as_tensor` | models/t3/t3.py:343, models/voice_encoder/voice_encoder.py:42 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.atleast_2d` | 7 sites: models/s3gen/flow.py×1, models/s3gen/s3gen.py×1, models/t3/modules/learned_pos_emb.py×1, models/t3/t3.py×1, mtl_tts.py×1, tts.py×1, tts_turbo.py×1 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.backends.cuda.sdp_kernel` | models/t3/modules/perceiver.py:94 | live | ✓✓✓ | starargs | **holds**; FutureWarning every generate (torch 2.14.1 `backends/cuda/__init__.py:676`) |
| torch | `torch.backends.mps.is_available` | 7 sites: example_for_mac.py×1, example_tts.py×1, example_vc.py×1, mtl_tts.py×1, tts.py×1, tts_turbo.py×1, vc.py×1 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.backends.mps.is_built` | mtl_tts.py:235, tts.py:171, tts_turbo.py:195, vc.py:65 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.bfloat16` | models/s3gen/decoder.py:28 | live | ✓✓✓ | - | **holds** |
| torch | `torch.bool` | 8 sites: models/s3gen/decoder.py×1, models/s3gen/transformer/attention.py×3, models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/encoder_layer.py×1 (dead), models/s3gen/transformer/encoder_layer.py×1, models/s3gen/utils/mask.py×1 | live+dead | ✓✓✓ | - | **holds** |
| torch | `torch.cat` | 36 sites: models/s3gen/decoder.py×1, models/s3gen/hifigan.py×1, models/s3gen/matcha/decoder.py×1, models/s3gen/matcha/text_encoder.py×4 (dead), models/s3gen/matcha/transformer.py×1, models/s3gen/transformer/attention.py×7, models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/embedding.py×2, models/s3gen/utils/intmeanflow.py×1, models/s3gen/xvector.py×2, models/t3/inference/t3_hf_backend.py×1, models/t3/modules/cond_enc.py×1, models/t3/t3.py×8, models/voice_encoder/voice_encoder.py×2, mtl_tts.py×1, tts.py×1, tts_turbo.py×1 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.clamp` | models/s3gen/flow.py:106, models/s3gen/hifigan.py:443, models/s3gen/utils/mel.py:16, models/s3tokenizer/s3tokenizer.py:165 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.clip` | models/s3gen/hifigan.py:405 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.complex` | models/s3gen/hifigan.py:408 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.concat` | models/s3gen/flow.py:161, models/s3gen/flow_matching.py:69 (dead), models/s3gen/flow_matching.py:70 (dead) | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.cos` | 9 sites: models/s3gen/flow_matching.py×1 (dead), models/s3gen/flow_matching.py×2, models/s3gen/hifigan.py×1, models/s3gen/s3gen.py×1, models/s3gen/transformer/embedding.py×4 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.cuda.is_available` | 6 sites: example_tts.py×1, example_vc.py×1, gradio_tts_app.py×1, gradio_tts_turbo_app.py×1, gradio_vc_app.py×1, multilingual_app.py×1 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.cuda.manual_seed` | gradio_tts_app.py:13, gradio_tts_turbo_app.py:71, multilingual_app.py:165 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.cuda.manual_seed_all` | gradio_tts_app.py:14, gradio_tts_turbo_app.py:72, multilingual_app.py:166 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.cumsum` | models/s3gen/hifigan.py:211 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.detach` | models/s3gen/matcha/text_encoder.py:410 (dead) | dead | ✓✓✓ | nosig | n/a (dead); resolves everywhere |
| torch | `torch.device` | 13 sites: example_for_mac.py×1, models/s3gen/flow.py×1, models/s3gen/hifigan.py×1, models/s3gen/utils/mask.py×2, mtl_tts.py×3, tts.py×2, tts_turbo.py×2, vc.py×1 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.distributions.uniform.Uniform` | models/s3gen/hifigan.py:29 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.div` | models/s3gen/utils/mask.py:84 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.dtype` | models/s3gen/decoder.py:26 | live | ✓✓✓ | - | **holds** |
| torch | `torch.einsum` | models/s3gen/matcha/text_encoder.py:140 (dead), models/t3/modules/perceiver.py:85, models/t3/modules/perceiver.py:90 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.empty` | models/s3gen/transformer/attention.py:135, models/s3gen/transformer/attention.py:255, models/s3gen/transformer/embedding.py:174, models/t3/modules/perceiver.py:188 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.exp` | 9 sites: models/s3gen/hifigan.py×2, models/s3gen/matcha/decoder.py×1, models/s3gen/matcha/transformer.py×2, models/s3gen/transformer/activation.py×1, models/s3gen/transformer/embedding.py×3 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.eye` | models/s3gen/utils/intmeanflow.py:13 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.flip` | models/s3gen/transformer/embedding.py:251 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.float` | models/s3gen/transformer/positionwise_feed_forward.py:108 | live | ✓✓✓ | - | **holds** |
| torch | `torch.float16` | models/s3gen/decoder.py:28 | live | ✓✓✓ | - | **holds** |
| torch | `torch.float32` | 9 sites: models/s3gen/decoder.py×1, models/s3gen/hifigan.py×1, models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/transformer/embedding.py×4, models/s3gen/xvector.py×1, models/voice_encoder/voice_encoder.py×1 | live+dead | ✓✓✓ | - | **holds** |
| torch | `torch.from_numpy` | 15 sites: models/s3gen/hifigan.py×1, models/s3gen/s3gen.py×2, models/s3gen/utils/mel.py×1, models/s3tokenizer/s3tokenizer.py×3, mtl_tts.py×2, tts.py×2, tts_turbo.py×2, vc.py×2 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.full` | models/voice_encoder/voice_encoder.py:46, models/voice_encoder/voice_encoder.py:178 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.full_like` | models/t3/modules/perceiver.py:38 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.hann_window` | models/s3gen/utils/mel.py:58, models/s3tokenizer/s3tokenizer.py:51 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.inference_mode` | 16 sites: models/s3gen/flow.py×1, models/s3gen/flow_matching.py×2, models/s3gen/hifigan.py×1, models/s3gen/matcha/flow_matching.py×1, models/s3gen/s3gen.py×3, models/t3/inference/t3_hf_backend.py×2, models/t3/t3.py×2, models/voice_encoder/voice_encoder.py×1, mtl_tts.py×1, tts.py×1, vc.py×1 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.int64` | models/s3gen/utils/mask.py:189 | live | ✓✓✓ | - | **holds** |
| torch | `torch.is_tensor` | 9 sites: models/s3gen/s3gen.py×1, models/s3tokenizer/s3tokenizer.py×1, models/t3/modules/cond_enc.py×1, models/t3/modules/learned_pos_emb.py×1, models/voice_encoder/voice_encoder.py×1, mtl_tts.py×1, tts.py×1, tts_turbo.py×1, vc.py×1 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.istft` | models/s3gen/hifigan.py:408 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.linalg.norm` | models/voice_encoder/voice_encoder.py:160, models/voice_encoder/voice_encoder.py:197 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.linspace` | models/s3gen/flow_matching.py:73 (dead), models/s3gen/flow_matching.py:223, models/s3gen/matcha/flow_matching.py:49, models/s3gen/s3gen.py:257 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.load` | 9 sites: example_for_mac.py×2, models/t3/modules/cond_enc.py×1, mtl_tts.py×3, tts.py×1, tts_turbo.py×1, vc.py×1 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.log` | models/s3gen/utils/mel.py:16, models/t3/modules/perceiver.py:36 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.log1p` | models/s3gen/matcha/text_encoder.py:255 (dead) | dead | ✓✓✓ | nosig | n/a (dead); resolves everywhere |
| torch | `torch.long` | models/t3/modules/perceiver.py:45, models/t3/modules/perceiver.py:46, models/t3/t3.py:255, models/t3/t3.py:305 | live | ✓✓✓ | - | **holds** |
| torch | `torch.manual_seed` | gradio_tts_app.py:12, gradio_tts_turbo_app.py:70, multilingual_app.py:163 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.matmul` | 7 sites: models/s3gen/matcha/text_encoder.py×2 (dead), models/s3gen/transformer/attention.py×4, models/s3gen/utils/mel.py×1 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.max` | models/s3gen/transformer/embedding.py:107, models/s3gen/utils/mel.py:50, models/t3/modules/perceiver.py:30 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.maximum` | models/s3tokenizer/s3tokenizer.py:166 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.mean` | models/s3gen/matcha/text_encoder.py:29 (dead), models/s3gen/matcha/text_encoder.py:30 (dead), models/voice_encoder/voice_encoder.py:195 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.min` | models/s3gen/utils/mel.py:49, models/t3/modules/perceiver.py:38 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.multinomial` | models/t3/t3.py:360, models/t3/t3.py:430, models/t3/t3.py:455 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.nn` | 20 sites: models/s3gen/decoder.py×1, models/s3gen/f0_predictor.py×1, models/s3gen/flow.py×1, models/s3gen/hifigan.py×1, models/s3gen/matcha/decoder.py×1, models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/matcha/transformer.py×1, models/s3gen/transformer/activation.py×1, models/s3gen/transformer/attention.py×1, models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/encoder_layer.py×1, models/s3gen/transformer/upsample_encoder.py×1, models/s3gen/utils/class_utils.py×1, models/s3gen/utils/intmeanflow.py×1, models/t3/inference/t3_hf_backend.py×1, models/t3/modules/cond_enc.py×1, models/t3/modules/learned_pos_emb.py×1, models/t3/modules/perceiver.py×1, models/t3/t3.py×1, models/voice_encoder/voice_encoder.py×1 | live+dead | ✓✓✓ | - | **holds** |
| torch | `torch.nn.BatchNorm1d` | models/s3gen/transformer/convolution.py:75, models/s3gen/xvector.py:138, models/s3gen/xvector.py:140 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.BatchNorm2d` | 5 sites: models/s3gen/xvector.py×5 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Conv1d` | 48 sites: models/s3gen/decoder.py×5, models/s3gen/f0_predictor.py×5, models/s3gen/hifigan.py×1, models/s3gen/matcha/decoder.py×4, models/s3gen/matcha/decoder.py×4 (dead), models/s3gen/matcha/text_encoder.py×13 (dead), models/s3gen/transformer/convolution.py×3, models/s3gen/transformer/subsampling.py×2, models/s3gen/transformer/upsample_encoder.py×3, models/s3gen/xvector.py×8 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Conv2d` | 12 sites: models/s3gen/transformer/subsampling.py×7, models/s3gen/xvector.py×5 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.ConvTranspose1d` | models/s3gen/hifigan.py:26, models/s3gen/matcha/decoder.py:144 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Dropout` | 17 sites: models/s3gen/matcha/text_encoder.py×5 (dead), models/s3gen/matcha/transformer.py×2, models/s3gen/transformer/attention.py×1, models/s3gen/transformer/embedding.py×3, models/s3gen/transformer/encoder_layer.py×1 (dead), models/s3gen/transformer/encoder_layer.py×1, models/s3gen/transformer/positionwise_feed_forward.py×1, models/s3gen/transformer/subsampling.py×2, models/t3/modules/perceiver.py×1 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.ELU` | 5 sites: models/s3gen/f0_predictor.py×5 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Embedding` | 7 sites: models/s3gen/flow.py×1, models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/transformer/subsampling.py×1, models/t3/modules/learned_pos_emb.py×1, models/t3/modules/perceiver.py×1, models/t3/t3.py×2 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.GELU` | models/s3gen/transformer/subsampling.py:134, models/s3gen/transformer/subsampling.py:136, models/s3gen/utils/class_utils.py:44 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.GroupNorm` | models/s3gen/decoder.py:235, models/s3gen/matcha/decoder.py:37, models/s3gen/matcha/decoder.py:353 (dead) | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Hardtanh` | models/s3gen/utils/class_utils.py:39 | live | ✓✓✓ | - | **holds** |
| torch | `torch.nn.Identity` | models/s3gen/utils/class_utils.py:55 | live | ✓✓✓ | - | **holds** |
| torch | `torch.nn.LSTM` | models/voice_encoder/voice_encoder.py:126 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.LayerNorm` | 16 sites: models/s3gen/decoder.py×1, models/s3gen/matcha/transformer.py×3, models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/encoder_layer.py×2 (dead), models/s3gen/transformer/encoder_layer.py×5, models/s3gen/transformer/subsampling.py×2, models/s3gen/transformer/upsample_encoder.py×1, models/t3/modules/perceiver.py×1 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Linear` | 34 sites: models/s3gen/decoder.py×1, models/s3gen/f0_predictor.py×1, models/s3gen/flow.py×2, models/s3gen/hifigan.py×1, models/s3gen/matcha/decoder.py×4, models/s3gen/matcha/decoder.py×1 (dead), models/s3gen/transformer/attention.py×5, models/s3gen/transformer/positionwise_feed_forward.py×3, models/s3gen/transformer/subsampling.py×5, models/s3gen/utils/intmeanflow.py×1, models/s3gen/xvector.py×1, models/t3/modules/cond_enc.py×2, models/t3/modules/perceiver.py×4, models/t3/t3.py×2, models/voice_encoder/voice_encoder.py×1 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Mish` | models/s3gen/decoder.py:57, models/s3gen/matcha/decoder.py:38, models/s3gen/matcha/decoder.py:49 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Module` | 82 sites: models/s3gen/decoder.py×2, models/s3gen/f0_predictor.py×1, models/s3gen/flow.py×3, models/s3gen/flow_matching.py×1, models/s3gen/hifigan.py×6, models/s3gen/matcha/decoder.py×6, models/s3gen/matcha/decoder.py×1 (dead), models/s3gen/matcha/flow_matching.py×1, models/s3gen/matcha/text_encoder.py×8 (dead), models/s3gen/matcha/transformer.py×3, models/s3gen/s3gen.py×1, models/s3gen/transformer/activation.py×2, models/s3gen/transformer/attention.py×1, models/s3gen/transformer/convolution.py×2, models/s3gen/transformer/embedding.py×3, models/s3gen/transformer/encoder_layer.py×3 (dead), models/s3gen/transformer/encoder_layer.py×5, models/s3gen/transformer/positionwise_feed_forward.py×4, models/s3gen/transformer/subsampling.py×8, models/s3gen/transformer/upsample_encoder.py×4, models/s3gen/xvector.py×9, models/t3/modules/cond_enc.py×1, models/t3/modules/learned_pos_emb.py×1, models/t3/modules/perceiver.py×4, models/t3/t3.py×1, models/voice_encoder/voice_encoder.py×1 | live+dead | ✓✓✓ | - | **holds** |
| torch | `torch.nn.ModuleList` | 37 sites: models/s3gen/decoder.py×9, models/s3gen/hifigan.py×8, models/s3gen/matcha/decoder.py×9 (dead), models/s3gen/matcha/text_encoder.py×6 (dead), models/s3gen/matcha/transformer.py×1, models/s3gen/transformer/positionwise_feed_forward.py×1, models/s3gen/transformer/upsample_encoder.py×2, models/s3gen/xvector.py×1 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.PReLU` | models/s3gen/xvector.py:136 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Parameter` | 14 sites: models/s3gen/hifigan.py×1, models/s3gen/matcha/text_encoder.py×2 (dead), models/s3gen/matcha/transformer.py×4, models/s3gen/transformer/activation.py×1, models/s3gen/transformer/attention.py×2, models/s3gen/transformer/embedding.py×1, models/t3/modules/perceiver.py×1, models/voice_encoder/voice_encoder.py×2 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.ReLU` | 15 sites: models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/positionwise_feed_forward.py×2, models/s3gen/transformer/subsampling.py×8, models/s3gen/utils/class_utils.py×1, models/s3gen/xvector.py×2 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.ReflectionPad1d` | models/s3gen/hifigan.py:378 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.SELU` | models/s3gen/utils/class_utils.py:42 | live | ✓✓✓ | - | **holds** |
| torch | `torch.nn.Sequential` | 17 sites: models/s3gen/decoder.py×1, models/s3gen/f0_predictor.py×1, models/s3gen/matcha/decoder.py×2, models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/transformer/subsampling.py×7, models/s3gen/xvector.py×5 | live+dead | ✓✓✓ | binds,starargs | **holds** |
| torch | `torch.nn.Sigmoid` | models/s3gen/xvector.py:212 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Tanh` | models/s3gen/hifigan.py:265, models/s3gen/utils/class_utils.py:40 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.Upsample` | models/s3gen/hifigan.py:329 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional` | 15 sites: models/s3gen/decoder.py×1, models/s3gen/flow.py×1, models/s3gen/flow_matching.py×1, models/s3gen/hifigan.py×1, models/s3gen/matcha/decoder.py×1, models/s3gen/matcha/flow_matching.py×1, models/s3gen/transformer/embedding.py×1, models/s3gen/transformer/upsample_encoder.py×1, models/s3gen/xvector.py×1, models/s3tokenizer/s3tokenizer.py×1, models/t3/modules/perceiver.py×1, models/t3/t3.py×1, models/voice_encoder/voice_encoder.py×1, mtl_tts.py×1, tts.py×1 | live | ✓✓✓ | - | **holds** |
| torch | `torch.nn.functional.avg_pool1d` | models/s3gen/xvector.py:223 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.nn.functional.cross_entropy` | models/t3/t3.py:220, models/t3/t3.py:221 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.embedding` | models/s3gen/transformer/embedding.py:113 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.glu` | models/s3gen/transformer/convolution.py:131 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.interpolate` | models/s3gen/matcha/decoder.py:153, models/s3gen/transformer/upsample_encoder.py:60 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.leaky_relu` | models/s3gen/hifigan.py:418, models/s3gen/hifigan.py:437, models/s3gen/transformer/upsample_encoder.py:88 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.max_pool1d` | models/s3gen/xvector.py:225 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.mse_loss` | models/s3gen/flow_matching.py:185, models/s3gen/matcha/flow_matching.py:112 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.normalize` | models/s3gen/flow.py:101, models/s3gen/flow.py:150 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.pad` | 12 sites: models/s3gen/decoder.py×1, models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/upsample_encoder.py×3, models/s3gen/utils/mel.py×1, models/s3tokenizer/s3tokenizer.py×2, mtl_tts.py×2, tts.py×2 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.relu` | 5 sites: models/s3gen/xvector.py×4, models/voice_encoder/voice_encoder.py×1 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.functional.scaled_dot_product_attention` | models/t3/modules/perceiver.py:95 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.nn.functional.softmax` | models/s3gen/matcha/text_encoder.py:245 (dead), models/s3gen/transformer/positionwise_feed_forward.py:106, models/t3/t3.py:429, models/t3/t3.py:454 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.init.constant_` | 8 sites: models/s3gen/decoder.py×4, models/s3gen/matcha/decoder.py×4 (dead) | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.init.kaiming_normal_` | 5 sites: models/s3gen/decoder.py×2, models/s3gen/matcha/decoder.py×2 (dead), models/s3gen/xvector.py×1 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.init.normal_` | models/s3gen/matcha/text_encoder.py:350 (dead) | dead | ✓✓✓ | binds | n/a (dead); resolves everywhere |
| torch | `torch.nn.init.xavier_uniform_` | 5 sites: models/s3gen/matcha/text_encoder.py×3 (dead), models/s3gen/transformer/attention.py×2 | live+dead | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.init.zeros_` | models/s3gen/xvector.py:415 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.utils.parametrizations.weight_norm` | models/s3gen/f0_predictor.py:16, models/s3gen/hifigan.py:28 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.nn.utils.remove_weight_norm` | models/s3gen/hifigan.py:27 (dead) | dead | ✓✓✓ | binds | n/a (dead); resolves everywhere |
| torch | `torch.no_grad` | models/s3gen/hifigan.py:200, models/s3gen/hifigan.py:275, models/s3gen/utils/intmeanflow.py:11, models/s3tokenizer/s3tokenizer.py:90 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.ones` | 16 sites: models/s3gen/hifigan.py×1, models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/matcha/transformer.py×2, models/s3gen/transformer/activation.py×1, models/s3gen/transformer/attention.py×3, models/s3gen/transformer/convolution.py×1, models/s3gen/transformer/encoder_layer.py×1 (dead), models/s3gen/transformer/encoder_layer.py×1, mtl_tts.py×2, tts.py×2, tts_turbo.py×1 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.ones_like` | models/t3/t3.py:259, models/t3/t3.py:407 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.pi` | models/s3gen/flow_matching.py:75 (dead), models/s3gen/flow_matching.py:170, models/s3gen/flow_matching.py:225, models/s3gen/s3gen.py:257 | live+dead | ✓✓✓ | - | **holds** |
| torch | `torch.pow` | models/s3gen/hifigan.py:30, models/s3gen/matcha/transformer.py:78, models/s3gen/transformer/activation.py:20 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.rand` | models/s3gen/flow_matching.py:168, models/s3gen/flow_matching.py:179, models/s3gen/matcha/flow_matching.py:105 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.randint` | models/s3gen/utils/mask.py:136, models/s3gen/utils/mask.py:144 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.randn` | models/s3gen/s3gen.py:316 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.randn_like` | 7 sites: models/s3gen/flow_matching.py×1 (dead), models/s3gen/flow_matching.py×2, models/s3gen/hifigan.py×2, models/s3gen/matcha/flow_matching.py×2 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.relu` | models/s3gen/matcha/text_encoder.py:89 (dead), models/s3gen/matcha/text_encoder.py:93 (dead), models/s3gen/matcha/text_encoder.py:273 (dead) | dead | ✓✓✓ | nosig | n/a (dead); resolves everywhere |
| torch | `torch.rsqrt` | models/s3gen/matcha/text_encoder.py:32 (dead) | dead | ✓✓✓ | nosig | n/a (dead); resolves everywhere |
| torch | `torch.save` | models/t3/modules/cond_enc.py:33, mtl_tts.py:145, tts.py:96, tts_turbo.py:101 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.sigmoid` | models/s3gen/transformer/activation.py:29 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.sin` | 10 sites: models/s3gen/hifigan.py×4, models/s3gen/matcha/transformer.py×1, models/s3gen/transformer/activation.py×1, models/s3gen/transformer/embedding.py×4 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.softmax` | models/s3gen/transformer/attention.py:113, models/s3gen/transformer/attention.py:119, models/t3/modules/perceiver.py:88, models/t3/t3.py:359 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.split` | models/s3gen/flow_matching.py:138, models/s3gen/transformer/attention.py:187, models/s3gen/transformer/attention.py:296 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.sqrt` | models/s3gen/utils/mel.py:80 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.stack` | models/s3gen/flow_matching.py:71 (dead), models/t3/t3.py:126, models/voice_encoder/voice_encoder.py:187, models/voice_encoder/voice_encoder.py:196 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.stft` | models/s3gen/hifigan.py:397, models/s3gen/utils/mel.py:66, models/s3tokenizer/s3tokenizer.py:156 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.sum` | models/s3gen/flow_matching.py:185, models/s3gen/matcha/flow_matching.py:113 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.tensor` | 9 sites: models/s3gen/transformer/embedding.py×1, models/s3gen/utils/intmeanflow.py×2, models/s3gen/utils/mel.py×1, models/t3/modules/learned_pos_emb.py×1, models/t3/t3.py×1, models/voice_encoder/voice_encoder.py×2, tts_turbo.py×1 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.topk` | models/s3gen/transformer/positionwise_feed_forward.py:103 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.transpose` | models/s3gen/decoder.py:45, models/s3gen/matcha/text_encoder.py:401 (dead) | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.unsqueeze` | 5 sites: models/s3gen/matcha/text_encoder.py×5 (dead) | dead | ✓✓✓ | nosig | n/a (dead); resolves everywhere |
| torch | `torch.utils.checkpoint` | models/s3gen/xvector.py:11 | live | ✓✓✓ | - | **holds** |
| torch | `torch.utils.checkpoint.checkpoint` | models/s3gen/xvector.py:271 | live | ✓✓✓ | binds | **holds** |
| torch | `torch.view_as_real` | models/s3gen/hifigan.py:401, models/s3gen/utils/mel.py:65 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.where` | models/s3gen/transformer/positionwise_feed_forward.py:112, models/t3/modules/perceiver.py:40 | live | ✓✓✓ | nosig | **holds** |
| torch | `torch.zeros` | 40 sites: models/s3gen/flow.py×2, models/s3gen/flow_matching.py×8, models/s3gen/hifigan.py×4, models/s3gen/matcha/text_encoder.py×1 (dead), models/s3gen/matcha/transformer.py×2, models/s3gen/s3gen.py×3, models/s3gen/transformer/activation.py×1, models/s3gen/transformer/attention.py×3, models/s3gen/transformer/convolution.py×2, models/s3gen/transformer/embedding.py×5, models/s3gen/transformer/encoder_layer.py×3 (dead), models/s3gen/transformer/encoder_layer.py×3, models/s3gen/utils/intmeanflow.py×1, models/t3/t3.py×2 | live+dead | ✓✓✓ | nosig | **holds** |
| torch | `torch.zeros_like` | models/s3gen/transformer/positionwise_feed_forward.py:109, models/t3/modules/cond_enc.py:71, models/t3/modules/perceiver.py:30 | live | ✓✓✓ | nosig | **holds** |
| torchaudio | `torchaudio` | 6 sites: example_for_mac.py×1, example_tts.py×1, example_tts_nano.py×1, example_tts_turbo.py×1, example_vc.py×1, models/s3gen/s3gen.py×1 | live | ✓✓✓ | - | **holds** |
| torchaudio | `torchaudio.compliance.kaldi` | models/s3gen/xvector.py:12 | live | ✓✓✓ | - | **holds** |
| torchaudio | `torchaudio.compliance.kaldi.fbank` | models/s3gen/xvector.py:50 | live | ✓✓✓ | binds | **holds** |
| torchaudio | `torchaudio.save` | 7 sites: example_for_mac.py×1, example_tts.py×3, example_tts_nano.py×1, example_tts_turbo.py×1, example_vc.py×1 | live | ✓✓✓ | binds | **breaks** on torchaudio >=2.9 without torchcodec (§3 R4); holds at 2.7.1 |
| torchaudio | `torchaudio.transforms.Resample` | models/s3gen/s3gen.py:44 | live | ✓✓✓ | binds | **holds** |
| tqdm | `tqdm.tqdm` | models/s3gen/flow_matching.py:19, models/t3/t3.py:8 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.AutoTokenizer` | tts_turbo.py:13 | live | ✓✓✓ | - | **holds** |
| transformers | `transformers.AutoTokenizer.from_pretrained` | tts_turbo.py:177 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.GPT2Config` | models/t3/t3.py:12 | live | ✓✓✓ | starargs | **holds** |
| transformers | `transformers.GPT2Model` | models/t3/t3.py:12 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.GenerationMixin` | models/t3/inference/t3_hf_backend.py:5 | live | ✓✓✓ | - | **holds** |
| transformers | `transformers.LlamaConfig` | models/t3/inference/t3_hf_backend.py:5, models/t3/t3.py:12 | live | ✓✓✓ | starargs | **holds** |
| transformers | `transformers.LlamaModel` | models/t3/inference/t3_hf_backend.py:5, models/t3/t3.py:12 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.LlamaPreTrainedModel` | models/t3/inference/t3_hf_backend.py:5 | live | ✓✓✓ | - | **holds** |
| transformers | `transformers.generation.logits_process.LogitsProcessorList` | models/t3/t3.py:13 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.generation.logits_process.MinPLogitsWarper` | models/t3/t3.py:13 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.generation.logits_process.RepetitionPenaltyLogitsProcessor` | models/t3/t3.py:13 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.generation.logits_process.TemperatureLogitsWarper` | models/t3/t3.py:13 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.generation.logits_process.TopKLogitsWarper` | models/t3/t3.py:13 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.generation.logits_process.TopPLogitsWarper` | models/t3/t3.py:13 | live | ✓✓✓ | binds | **holds** |
| transformers | `transformers.modeling_outputs.CausalLMOutputWithCrossAttentions` | models/t3/inference/t3_hf_backend.py:6 | live | ✓✓✓ | binds | **holds** |
