# Bugs

Found in a read-only audit on 2026-10-18 (branch `relax-dependency-pins`, commit `f600339`).
Nothing was executed: system Python has no torch, and only the multilingual weights are cached.
Items 1, 6 and 7 are traced through the code, not reproduced. Numbers match the audit report;
tech debt items are in `TECH_DEBT.md`.

Assumed load: one caller at a time per model instance (`generate` mutates `self.conds`);
demos run locally or behind a Gradio share link with a few users.

## 1. English model crashes when `cfg_weight=0`

- Where: `src/chatterbox/tts.py:237-243`, `src/chatterbox/models/t3/t3.py:305-313`
- What: CFG (classifier-free guidance) runs the model on the text and on an empty prompt, then
  blends the two. `cfg_weight` is the blend strength.
- Problem: `tts.py` doubles the text batch only when `cfg_weight > 0`, but `T3.inference` always
  appends a batch of 2 start-of-speech embeds. With 0, `torch.cat` joins batch 1 to batch 2 and
  raises a size-mismatch `RuntimeError`. Reachable from the `gradio_tts_app.py` slider (min 0.0),
  and README line 164 tells users to set 0.
- Fix: delete the `if` on line 237 so the batch is always doubled, like `mtl_tts.py:317`.
  With weight 0 the blend returns the plain result.
- If skipped: users following the README get a crash.

## 2. Voice-conversion demo gives one user's voice to the next user

- Where: `gradio_vc_app.py:9-14`, `src/chatterbox/vc.py:88-91`
- What: the demo shares one global model; its label says "if none, the default voice is used".
- Problem: `set_target_voice` overwrites `model.ref_dict` and nothing restores it. After user A
  uploads a target voice, user B with no target gets A's voice.
- Fix (in the demo, not the library API): save `DEFAULT_REF = model.ref_dict` at startup and set
  `model.ref_dict = DEFAULT_REF` in `generate` when no target is given.
- If skipped: wrong output, and one person's cloned voice reaches a stranger.

## 3. Chinese breaks without a network, and the mapping downloads twice

- Where: `src/chatterbox/models/tokenizers/tokenizer.py:167-188`, `:260`
- What: Chinese characters are converted to Cangjie codes (key spellings of each character).
  Only the codes are in the model vocabulary.
- Problem: `hf_hub_download(cache_dir=model_dir)` uses the model's own snapshot folder as a
  cache, creating a nested `models--ResembleAI--chatterbox` inside it (seen on this machine).
  Offline or on a read-only folder the error is swallowed as a warning. Checked: raw characters
  like `的我是中` are not in `grapheme_mtl_merged_expanded_v1.json`, so every Chinese character
  becomes `[UNK]`.
- Fix: read `model_dir / "Cangjie5_TC.json"` directly (`from_pretrained` already downloads it,
  `mtl_tts.py:247`) and raise if it is missing instead of warning.
- If skipped: Chinese output is garbage with only a log warning, e.g. in an offline
  ebook2audiobook install.

## 4. `gradio==6.8.0` is a hard library dependency (fixed: now the `demo` extra)

- Where: `pyproject.toml:28`
- Problem: nothing in `src/` imports gradio; only the demo apps do. ebook2audiobook needs
  `gradio==6.29.1`, so the two cannot be installed together. This is the conflict this branch
  exists to fix.
- Fix: move it to `[project.optional-dependencies] demo = ["gradio>=6.8"]` and note
  `pip install -e .[demo]` in the README for the demos.
- If skipped: the dependency migration fails at its main goal.

## 5. English and Turbo demos load a full model per visitor

- Where: `gradio_tts_app.py:19-21,73`, `gradio_tts_turbo_app.py:76-79,166`
- What: `demo.load(load_model)` stores a model in `gr.State`, which Gradio keeps per browser
  session.
- Problem: every tab or visitor loads its own multi-GB copy onto the GPU and keeps it until the
  session ends. Both demos launch with `share=True` (public link), so a few visitors exhaust GPU
  memory.
- Fix: load once at module level like `gradio_vc_app.py`, together with the bug 2 fix: save
  `model.conds` at startup and restore it when no reference audio is given. Without that, a
  shared model leaks voices here too.
- If skipped: the demo dies after about 3 to 5 visitors.

## 6. Multilingual path does not drop out-of-range speech tokens

- Where: `src/chatterbox/mtl_tts.py:339` (compare `tts.py:262`, `tts_turbo.py:308`)
- What: T3 samples from 8194 speech tokens; S3Gen's `input_embedding` knows only 6561
  (`src/chatterbox/models/s3gen/flow.py:76`).
- Problem: `tts.py` and Turbo filter `< 6561`; `mtl_tts.py` only slices at start/stop tokens.
  Any other high token (more likely at the high temperatures the demo allows) raises an index
  error in `input_embedding`; `flow.py:164` only logs first.
- Fix: filter once in `S3Token2Wav.inference`, which already has an unused
  `drop_invalid_tokens=True` argument (`src/chatterbox/models/s3gen/s3gen.py:339-345`): apply
  `speech_tokens[speech_tokens < SPEECH_VOCAB_SIZE]`. Then delete the copies in `tts.py` and
  `tts_turbo.py` and the unused helper at `s3gen.py:36-38`.
- If skipped: rare, random-looking crashes in the multilingual model only.

## 7. Long text crashes; long audio is cut off silently

- Where: `src/chatterbox/models/t3/t3.py:77,118`, `tts.py:249`, `mtl_tts.py:328`, `t3.py:393`
- What: the text position table has 2050 entries (`max_text_tokens + 2`); generation is capped
  at 1000 speech tokens (40 s).
- Problem: text over about 2048 tokens raises an index error deep in torch. Speech needing more
  than 40 s stops mid-sentence with no warning. `gradio_tts_app.py` says "max chars 300" but
  does not truncate, unlike `multilingual_app.py:239`.
- Fix: check `text_tokens.size(-1) <= hp.max_text_tokens` and raise a clear error; log a warning
  when the sampling loop ends without a stop token.
- If skipped: audiobook chapters end early with nothing in the log.

## 8. One checkpoint load skips `weights_only=True`

- Where: `src/chatterbox/vc.py:50`
- What: `torch.load` without `weights_only=True` unpickles the file; a crafted file can run code.
- Problem: torch 2.6+ defaults to the safe mode, older torch does not, and the migration note
  considers 2.4/2.5. It also breaks the rule in `CLAUDE.md`.
- Fix: add `weights_only=True`.
- If skipped: loosening the torch pin opens a code-execution path through `from_local(ckpt_dir)`.
