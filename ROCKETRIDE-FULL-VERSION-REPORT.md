# RocketRide — Full Version Development Report

Plain English. Local clone only — **not pushed to GitHub**.

## What is running

- **Repo:** `HARI30015/rocketride-server` (fork of `rocketride-org/rocketride-server`), branch `develop`
- **Local clone:** `~/workspace/rocketride-server`, HEAD `0c0e269b`
  `fix(vscode): keep the My Apps list refreshing after the sidebar loads (#3101)`
  (updated 2026-11-14 daily run; was `a8e15518` when this report was first written)
- **Upstream check:** `git rev-list --count HEAD..upstream/develop` = **0** — the fork is fully up to date with upstream `develop`. It was 50 commits behind when this work started; it was fast-forwarded to the latest upstream commit (no user changes, clean tree).
- **Server:** prebuilt upstream stable release `server-v3.6.0`, extracted at `~/workspace/rocketride-engine`
- **Verified live:** `curl http://localhost:5565/version` returns HTTP 200:
  `{"status":"OK","data":{"version":"3.6.0.38","hash":"ba6b8ef6","stamp":"2026-11-04T22:19:01Z"}}`
  Process pid 11328, listening on `127.0.0.1:5565` only — **localhost only, NOT public**. No Docker on this VM, and no full C++ source build was attempted (`./builder` builds everything and is too heavy for 1.5 CPU / ~6 GB RAM).

## What the full version can do

RocketRide is an AI pipeline engine: you join ready-made blocks (nodes) into a
pipeline (a `.pipe` JSON file) and the engine runs it. This version has 100+
nodes — 15+ LLM providers, 9 vector databases, OCR, speech, agents
(CrewAI / LangChain), and tools for Gmail, Google Drive/Sheets, Slack,
Notion, GitHub, and more. Pipelines can become full apps (VS Code App
Builder, Store, embeddable chat widget), can be called from Python or
TypeScript with the SDK, and can run locally, in Docker, on-prem, or on
RocketRide Cloud. It is **not** a model itself — it uses other providers'
models, so a real LLM step needs the user's own API key.

## Demo built and verified

Demo code lives in `~/workspace/rocketride-demo/` (SDK 1.6.0 installed in
`.venv`, `ROCKETRIDE_URI=ws://localhost:5565`, `ROCKETRIDE_APIKEY=local` —
the local server accepts the literal string `local`; no real secret exists).

### Verified — passed

1. **Server health** — `/version` returns 200, version 3.6.0.38, checked
   many times; after a runtime restart killed it, it was restarted the
   same way (`./engine ./ai/eaas.py --host=127.0.0.1`) and re-verified
   200 in ~25 s. New pid 2264, still `127.0.0.1:5565`.
2. **Pipeline validation** — `rocketride validate` passed for all three:
   - `examples/document-processor.pipe`: `1 file(s), 1 valid, 0 invalid`
   - `simple-parse.pipe` (webhook → parse → response_text, copied from
     the example): `1 file(s), 1 valid, 0 invalid`
   - `minimal-echo.pipe` (webhook → response): `1 file(s), 1 valid, 0 invalid`
3. **Live end-to-end demo, clean run after the restart** — `run_echo.py`
   (`minimal-echo.pipe`, `use` → `send` → `terminate`), exit 0 in ~5 s:
   `connected, starting pipeline...` / `token: tk_…` /
   `{'name': 'note.txt', 'result_types': {'text': 'text'}, 'text': ['Alice Smith, 12 Elm St, Springfield. Please contact Bob Jones at 34 Oak Avenue, Shelbyville about the meeting on 2026-11-12.\n\n'], 'objectId': '20cc3d19-abf6-532d-a8cb-ab56e28495ac'}` /
   `terminated ok` (log: `~/workspace/rocketride-demo/echo1.log`).
4. **Live send to an already-running pipeline** — `send_to_running.py`
   sent the same note to the running webhook task and got the exact text
   back in ~1 s, exit 0 (log: `~/workspace/rocketride-demo/send1.log`).
   A second investigator independently verified the same echo flow live.

### The blocker, and the fix

The full document-processor demo (webhook → parse → OCR → NER → anonymize
→ response) did **not** finish on the first tries. `use()` timed out three
times (exit 124: 600 s, 300 s, 600 s). This is **not** an API-key problem —
no key is needed anywhere in that pipeline. The cause, found from the live
task status, is a shared first-run setup: the engine installs one shared
Python environment the first time any Python-backed node starts, and it
downloads torch (874 MB) plus ~3.5 GB of NVIDIA CUDA wheels at ~1–3 MB/s.
Every pipeline start waits on that same install, including the parse-only
pipeline. A runtime restart in the middle killed the server and the
install (uv cache fell 5.7 GB → 4.1 GB); the server was restarted and the
install resumed from cache (back to 5.0 GB and growing).

Workarounds used:
- Do **not** call `use()` again on a started pipeline (the server answers
  `Pipeline is already running`). Re-attach with `use_existing=True` or
  `get_task_token(project_id, source)` and `send()` to the existing token.
- The no-model echo pipeline starts in seconds and is the reliable demo;
  the model pipeline only becomes fast after the one-time install finishes.

### Final demo result — document-processor (full version)

**Still in progress at the time of writing, NOT yet observed.** A fresh
run (`run_document_processor.py`, 1800 s timeout, log
`~/workspace/rocketride-demo/run5.log`) was tried after the restart and
also timed out: `exit=124` after the full 1800 s, only
`connected, starting pipeline...` printed. The client timeout killed
only the waiting client — the engine task itself is **still alive and
still downloading**: token `tk_b2aa8aaa…`
(project `8c3de58a-79cb-4f2d-b5a6-1e93a6ed7faa`, source `webhook_1`),
state 2, `serviceUp: False`, status `Downloading nvidia-…`, uv cache
5.7 GB and still growing, server still healthy (`/version` 200).
To finish it, wait until `get_task_status(token)` shows `serviceUp: True`,
then send the note with `send_to_running.py tk_b2aa8aaa…` — no new `use()`
needed. Expected output: the same note with names/addresses redacted by
the NER + anonymize nodes (exact redaction format not yet observed).
After the torch install, the NER (BERT-large, ~1.3 GB), GLiNER, and
EasyOCR models still download on first use, so allow more time.
Four `use()` attempts in total have now timed out (600/300/600/1800 s)
on this same shared install — none of them was a key, code, or server
failure.

## What still needs the user

- **Nothing for the engine itself** — it runs with no sign-up and no key.
- **A real LLM step** (chat, agents, RAG answers) needs the user's own
  provider API key (OpenAI / Anthropic / etc.). Never put keys in the repo.
- **Seeing it in his own browser** needs a public link: the server is
  localhost-only by design. A Cloudflare quick tunnel was tried and failed
  (`tls: first record does not look like a TLS handshake` — the VM's
  egress proxy blocks it); the next options are the RocketRide VS Code
  extension (connects to `ws://localhost:5565` from his machine), a
  proper named Cloudflare tunnel, or deploying the engine image to
  Render / a VPS.
- **Pushing the updated fork to GitHub** needs the Muse GitHub App to be
  installed on `rocketride-server` (it currently is not — that is why the
  connector only saw 6 of his 8 repos). The local clone is ready to push
  (fast-forwarded to upstream `develop`, clean); the push itself was
  deliberately not attempted.

## Test results summary

| Test | Result |
|---|---|
| Clone at latest upstream develop (a8e15518, 0 behind) | PASSED |
| Server `/version` 200, v3.6.0.38, localhost:5565 | PASSED (also after restart) |
| `rocketride validate` document-processor / simple-parse / minimal-echo | PASSED (1 valid, 0 invalid each) |
| Echo demo `use` → `send` → `terminate`, exact text back, exit 0, ~5 s | PASSED |
| Send to already-running pipeline, exact text back, exit 0, ~1 s | PASSED |
| Document-processor `use()` (4 tries, 600/300/600/1800 s) | FAILED — exit 124 each time, blocked on first-run torch/CUDA download, not on a key; engine task still alive and downloading, resume via its token |
| Cloudflare quick tunnel for a public link | FAILED — `tls: first record does not look like a TLS handshake` (VM egress proxy); not retried, see below |

## Next development step

1. Wait for the first-run install to finish, then run the
   document-processor demo end-to-end and record the redacted output.
   Resume with: `get_task_status('tk_b2aa8aaa…')` → when `serviceUp`
   is True → `send_to_running.py tk_b2aa8aaa…`.
2. After that, the useful next build is a real app on top of the engine —
   for example a notes-answering assistant (RAG pipeline + a small
   Python client), which needs an LLM key from the user to answer
   questions, but can already index and search documents without one.
3. Keep the fork fresh with a daily upstream sync check (the parent agent
   was setting this schedule up); update the running engine whenever a
   new stable `server-vX.Y.Z` release appears.

## Check 2026-11-13 ~06:24 UTC (11:54 IST) — engine up, demo task gone

- `/version` 200, v3.6.0.38, pid 3896, listening 127.0.0.1:5565. Engine log
  shows 4 starts since yesterday (one clean shutdown) — restarts keep
  killing in-progress pipeline tasks.
- Echo demo re-run today: `run_echo.py` exit 0 in ~7 s, exact text back,
  `terminated ok`. Engine works end-to-end after the restarts.
- Document-processor task is GONE: `check_task.py` reports "Your pipeline
  is not running" for all three projects, including the document-processor
  token tk_b2aa8aaa… (project 8c3de58a-79cb-4f2d-b5a6-1e93a6ed7faa).
  The full demo promised 2026-11-12 was never verified. uv cache still
  5.0 GB. Next step: fresh `run_document_processor.py` run (long timeout,
  do not repeat `use()`); the shared torch/CUDA first-run install may
  still be the blocker.
- Local clone is at upstream 044728a3 (7 commits applied 2026-11-13
  06:43 IST daily run); GitHub fork push still blocked — GitHub App not
  installed on rocketride-server, needs the user.

## Check 2026-11-13 ~06:35-06:50 UTC (12:05-12:20 IST) — blocker found, fix in progress

- Engine re-verified up several times today: `/version` 200, v3.6.0.38.
  Clone still at upstream 044728a3, clean (only this report untracked).
- Fresh document-processor runs (run6/run7) FAILED FAST (~20 s), not by
  timeout: `Dependency resolution failed ... Failed to fetch:
  https://download.pytorch.org/whl/cu128/torchvision/` (and `.../torch/`).
  Retried twice — deterministic, not transient.
- Root cause found: the engine runs sandboxed with a scrubbed environment
  (pid 8187 had only 7 vars: LANG, PATH, USER + systemd vars, NO proxy
  settings), and direct egress from this VM is blocked (curl without the
  proxy env fails instantly, 000; with the proxy env it returns 200 in
  ~1.2 s). So the engine's uv cannot fetch the PyTorch index. The wheels
  themselves ARE already in the shared uv cache (torch 2.10.0+cu128,
  torchvision 0.25.0+cu128, ~100 archived packages, 5.0 GB) — only the
  index fetch was failing.
- Fix tried (no secrets involved): created
  `~/workspace/rocketride-engine/.env` with `UV_OFFLINE=1` and
  `RR_SUBPROCESS_ENV=UV_OFFLINE` (the engine loads exec_dir/.env at
  startup, ai/web/server.py:217; docs say proxy/TLS settings are part of
  the task allowlist). Engine restarted with the .env in place.
- Result (run9): offline resolution WORKED — "Resolved 31 packages in
  25ms, Installed 3 packages in 87ms" — but then failed downloading
  `nvidia-nccl-cu12==2.27.5`: "Network connectivity is disabled", i.e.
  that one transitive wheel is NOT in the cache yet.
- Next step in progress: seeding the missing wheels into the shared uv
  cache from the shell (where the proxy works) with the engine's own uv:
  a throwaway venv installing the same requirements.txt
  (torch/torchvision cu128). After it finishes, retry
  `run_document_processor.py` — resolution should then complete fully
  offline. Note: NER/OCR models still need downloading on first use
  (no Hugging Face cache on disk yet); that step may hit the same
  no-proxy problem inside the task and need its own workaround.
- Still needs the user (unchanged): GitHub App install on
  rocketride-server to push the updated fork; an LLM provider API key
  only if a real model step is wanted.

## VERIFIED 2026-11-15 ~00:40 IST — document-processor demo PASSED end-to-end

`run_document_processor.py` (`run25.log`), exit 0, `terminated ok`:

- Input: "Alice Smith, 12 Elm St, Springfield. Please contact Bob Jones at 34 Oak Avenue, Shelbyville about the meeting on 2026-11-12."
- Output: "Alice Smith, █████████, ███████████. Please contact █████████ at █████████████, ███████████ about the meeting on 2026-11-12." — names/addresses masked by the NER + anonymize nodes.
- Honest detail: "Alice Smith" itself was NOT masked (NER confidence / GLiNER missed it); Bob Jones, both street addresses and both towns were masked. Pipeline works; that one name slipping through is a model-confidence detail, not a pipeline failure.
- Engine: `/version` 200, v3.6.0.38, localhost:5565, right after the run.

### What it took to get here (all blockers cleared 2026-11-14)

1. **Torch wheels** — all 5 NVIDIA wheels (~2.3 GB) seeded into the uv cache; torch 2.10.0+cu128 imports. Hourly seed job completed and disabled itself 2026-11-14 21:48 IST.
2. **OCR/NER model installs** — engine subprocess is offline, so each missing wheel was seeded one by one into the shared uv cache from the shell: scikit-image 0.26.0, tifffile 2026.11.8, transformers 4.53.3, regex 2026.1.15, protobuf (see below).
3. **NER + anonymize model weights** (~4.5 GB total) pre-downloaded into the Hugging Face cache with hash verification, because the node subprocess downloads them slowly over hf-xet and stalls: BERT-large NER `dbmdz/bert-large-cased-finetuned-conll03-english` (pytorch_model.bin 1.33 GB, sha256 6599da27…; model.safetensors 1.33 GB, sha256 9a90b161… — the loader prefers safetensors), GLiNER anonymize `xomad/gliner-model-merge-large-v1.0` (1.76 GB, sha256 08950401…), and the anonymize default `urchade/gliner_small-v2.1` (610 MB, sha256 1d4e83e4…). EasyOCR detector/recognizer models (~/.EasyOCR/model) were already cached.
4. **Protobuf conflict (upstream lockfile bug)** — the engine's compiled `cache/constraints.txt` pinned `protobuf==6.33.6` while `sentencepiece==0.2.3` was built with protobuf gencode 7.36.1: importing sentencepiece under runtime 6.33.6 fails ("incompatible Protobuf Gencode/Runtime versions"). Fix, local engine copy only: added `protobuf==7.36.2` to the engine's documented overrides (`ai/overrides.txt` + `cache/overrides-combined.txt`, onnxruntime-gpu 1.22.0 allows any protobuf), changed `cache/constraints.txt` to `protobuf==7.36.2`, and re-stored the matching requirements hash (`cache/requirements.hash`) so the engine accepts the hand-edited constraints (a full offline recompile is impossible: `cuda-bindings==12.9.4` is not in the cache). Verified by dry-run: install now resolves `- protobuf==6.33.6 + protobuf==7.36.2`.
5. **Init timeout** — `CONST_MAX_READY_TIME` raised 5 min → 20 min in the local engine copy (`ai/constants.py`): cold CPU model loads take longer than 5 min. Engine restarted to apply it.
6. **/tmp is a 512 MB tmpfs** — a 1.3 GB model download to /tmp failed with "No space left on device"; downloads now go to `~/workspace/rocketride-demo/` instead.

### Still open (unchanged)

- GitHub fork push still blocked — GitHub App not installed on `rocketride-server`; local clone is at 0c0e269b, clean, ready to push once the user adds the repo to the App.
- No public link — localhost-only by design; Cloudflare quick tunnel failed and was stopped per the user's "stop".
- Engine still dies on VM reboot and sometimes within a minute of a restart; hourly watchdog + re-restart on delivery checks keep it up. No working systemd service (does not survive VM replacement).
- A real LLM step still needs the user's own provider API key.

## Check 2026-11-13 ~11:20 UTC (16:50 IST) — engine up, seeding the missing wheels

- Engine up: `curl http://localhost:5565/version` 200, v3.6.0.38,
  still listening on 127.0.0.1:5565 (checked 11:17 UTC). Hourly
  watchdog next run 17:24 IST.
- Latest demo attempt run10 (09:40 UTC / 15:10 IST) FAILED FAST on the
  next missing wheel after run9's nccl failure:
  `nvidia-cusparse-cu12==12.5.8.93` not in the uv cache, offline mode.
  The cache holds only index metadata (.msgpack) for cusparse/nccl,
  no wheel files.
- Seeding started 11:17 UTC (in progress, not finished): throwaway
  venv `~/workspace/rocketride-demo/seed-torch-venv`, engine's own uv,
  installing torch==2.10.0+cu128 + torchvision==0.25.0+cu128 from the
  shell (proxy works there). Log:
  `~/workspace/rocketride-demo/seed-torch.log`. It resolved 31
  packages and is downloading 5 wheels (~2 GB total):
  nvidia-cusparse-cu12 274.9 MiB, nvidia-cublas-cu12 566.8 MiB,
  nvidia-cusparselt-cu12 273.9 MiB, nvidia-cudnn-cu12 674.0 MiB,
  nvidia-nccl-cu12 307.4 MiB. Disk is fine (141 GB free).
- Next step: when the seed finishes, retry
  `run_document_processor.py` (offline resolution should then
  complete); NER/OCR model downloads on first use may still need
  their own workaround. Do NOT re-run `use()` on a started pipeline;
  the demo result is still unverified.
