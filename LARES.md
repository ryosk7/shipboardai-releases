# Shipboard AI runbook for Lares

This file is written for **Lares** (the Olares AI assistant) to install, update and check **Shipboard AI** with
`olares-cli`, and to bring the device to the **recommended setup** (section 1). A person can read it too.

> 日本語の利用者へ：Lares に「https://github.com/ryosk7/shipboardai-releases を見てアップデートして」と頼んでください。
> Lares がこの手順書に沿って状態を確認し、足りないモデルの導入と Shipboard AI の更新を提案します。
> 端末を変える操作は、Lares が内容を示したあとに、あなたが承認してから行われます。

## 0. Rules (read first)

1. **Report in the user's language** (Japanese for this vessel). Show the exact command before running anything that
   changes the device, and wait for the user's approval. One approval may cover a whole section's commands if the user
   says so. Read-only commands (`list`, `get`, `curl` to GitHub) need no approval.
2. **Never** uninstall Shipboard AI or any other app, never delete uploaded charts, never stop apps you were not asked
   to stop, never change GPU / accelerator settings, never edit app env values. In particular, **never stop or
   uninstall EmbeddingGemma** (`embeddinggemmav3`) or another embedding app that is running: Shipboard AI may still be
   searching with it (section 9). If a step seems to need one of these, stop and ask.
3. **Stop at the first failure.** Do not retry with different flags or another source. Report the command, its output
   and the state from `olares-cli market list --mine`.
4. **Idempotent**: if a step's goal is already met (app running, version already installed), skip it and say so.
5. Use only the package source in section 2. Do not download Shipboard AI from anywhere else.
6. Installing apps and uploading custom packages needs an **Olares administrator** and Lares' **write** (or full)
   permission. If a command is denied, stop and tell the user which permission is missing.
7. **No downgrades.** Shipboard AI is never installed at a version older than the installed one (Olares refuses it as
   well). If an update misbehaves, report it and wait for a newer release.

## 1. The recommended setup

Bring the device to this setup. "Required" and "recommended" apps are both installed by default; "optional" only when
the user asks.

| Olares app id | Title in Market | Level | Role |
|---|---|---|---|
| `router` | Router | required | AI gateway; Shipboard AI calls every model through it. Shipboard AI cannot install without it. |
| `llamacppqwen3827bggufv3` | Qwen3.8-27B (llama.cpp) | required | Chat model: answers, checks, report reviews, troubleshooting. Needs an NVIDIA GPU with about 23 GiB VRAM. |
| `llamacppllmbasev3` cloned as "Qwen3 Embedding 8B (llama.cpp)" (section 4) | llama.cpp Engine Base, Qwen3 Embedding 8B Q4_K_M on the CPU | required | Embedding model for document search. Router lists it as `Olares/Qwen/Qwen3-Embedding-8B-GGUF:Q4_K_M`. Runs on the CPU (about 1 s per question, ~6 GiB RAM) so questions never switch the GPU away from Qwen3.8. |
| `bgererankerv2m3` | BGE-Reranker-v2-m3 | recommended | Reranks search results (Intel iGPU, compute mode `intel`). |
| `dotsocr` | dots.ocr | recommended | Reads scanned pages when documents are registered (NVIDIA, compute mode `nvidia`, about 11 GiB VRAM, only while documents are processed). |
| `embeddinggemmav3` | EmbeddingGemma | keep if installed | The embedding model of older setups. Do not install it for a new setup; if it is installed, keep it running (section 9). |
| `levv1` | Lev System One | optional | Fast decision model; Shipboard AI records its decisions. Not required to answer. Install only when the user asks. |
| `shipboardai` | Shipboard AI | — | The app itself, installed from an uploaded package (source `upload`). |

Olares must be **1.12.7 or later**.

## 2. Package source

- Releases: `https://github.com/ryosk7/shipboardai-releases/releases`
- Latest release (JSON): `https://api.github.com/repos/ryosk7/shipboardai-releases/releases/latest`
  - `tag_name` is `v<version>` (for example `v0.4.33`).
  - The asset `shipboardai-<version>.tgz` is the package; its `browser_download_url` is the download link.

## 3. Check (read-only) — always run this first

```sh
olares-cli market list --mine -o json
curl -s https://api.github.com/repos/ryosk7/shipboardai-releases/releases/latest
```

From the first output, read `state` and `version` of every app in section 1. The Qwen3 Embedding app is a clone: its
`title` contains `Qwen3 Embedding 8B (llama.cpp)` and its `name` looks like `llamacppllmbasev314c33e`. An older
`ollamallmbasev3…` row ("Qwen3 Embedding 8B (Ollama)") is no longer used and may stay stopped. From the second output,
read the latest version (`tag_name` without the leading `v`).

Report a table like this, then propose the next sections in order:

| App | Level | Installed | State | Note |
|---|---|---|---|---|
| Router | required | 1.1.3 | running | ok |
| Qwen3.8-27B | required | 1.0.21 | running | ok |
| Qwen3 Embedding 8B (llama.cpp) | required | — | — | missing → section 4 |
| BGE-Reranker-v2-m3 | recommended | 1.0.8 | running | ok |
| dots.ocr | recommended | 1.0.17 | stopped | → section 5 |
| EmbeddingGemma | keep | 1.1.5 | running | keep running |
| Shipboard AI | — | 0.4.32 (latest 0.4.33) | running | update → section 7 |

Decide, in this order:
1. Any state ending in `Failed` (for example `installFailed`, `upgradeFailed`) → stop and report; do not try to fix it.
2. A required or recommended app is missing → section 4.
3. A required or recommended app, or a running-before EmbeddingGemma, is `stopped` → section 5.
4. Shipboard AI is missing → section 6.
5. Shipboard AI is older than the latest release → section 7.
6. Everything in section 1 is running and Shipboard AI is the latest → report "準備完了", then section 9.

## 4. Install a missing model app (needs approval)

Install the missing required apps first, in this order, then the missing recommended ones:

```sh
# required
olares-cli market install router --watch
olares-cli market install llamacppqwen3827bggufv3 --compute-mode nvidia --watch --watch-timeout 1h
olares-cli market clone llamacppllmbasev3 --title "Qwen3 Embedding 8B (llama.cpp)" --compute-mode nvidia \
  --env "MODEL_SOURCE=hf://Qwen/Qwen3-Embedding-8B-GGUF --include Qwen3-Embedding-8B-Q4_K_M.gguf" \
  --env "MODEL_NAME=Qwen/Qwen3-Embedding-8B-GGUF:Q4_K_M" \
  --env MODEL_MODE=embedding --env MODEL_SUPPORTS=none \
  --env "ENGINE_ARGS=-c 2048 -ub 2048 -b 2048 --pooling last -ngl 0 -dev none" \
  --watch --watch-timeout 1h
# The engine flags live on the model card, which wins over ENGINE_ARGS after the first start; set them there too:
olares-cli router model spec edit 'Olares/Qwen/Qwen3-Embedding-8B-GGUF:Q4_K_M' \
  --engine-args "-c 2048 -ub 2048 -b 2048 --pooling last -ngl 0 -dev none" --yes

# recommended
olares-cli market install bgererankerv2m3 --compute-mode intel --watch --watch-timeout 30m
olares-cli market install dotsocr --compute-mode nvidia --watch --watch-timeout 1h
```

- The Qwen models and dots.ocr download tens of GB; a long wait is normal.
- If a compute mode above is refused (the device has no Intel iGPU, for example), stop and report the modes the error
  lists; do not pick another mode yourself. The user can install that app from Market in the browser instead.
- If an install asks for a GPU binding you cannot answer from this file, stop and ask the user to install that app
  from Market in the browser.

## 5. Resume a stopped app (needs approval)

```sh
olares-cli market resume <app id> --watch
```

Only resume apps listed in section 1. If the resume needs a `--compute-binding`, stop and ask the user to resume it from
Market → My Olares in the browser.

## 6. Install Shipboard AI (needs approval)

Preconditions: `router`, `llamacppqwen3827bggufv3` and the Qwen3 Embedding app are `running` (section 3).

```sh
curl -fL -o shipboardai-<version>.tgz \
  https://github.com/ryosk7/shipboardai-releases/releases/download/v<version>/shipboardai-<version>.tgz
olares-cli market upload shipboardai-<version>.tgz
olares-cli market install shipboardai -s upload --version <version> --watch
```

Success: `shipboardai` shows `state: running` and `version: <version>` with `source: upload`. Then tell the user to open
Shipboard AI from the Launchpad; the administrator finishes the setup in the app (vessel, crew, AI connection check).
Shipboard AI picks its models by itself from what Router lists; nothing is configured by hand.

## 7. Update Shipboard AI (needs approval)

Preconditions: `shipboardai` is `running` or `stopped`, and its version is older than the latest release.

```sh
curl -fL -o shipboardai-<version>.tgz \
  https://github.com/ryosk7/shipboardai-releases/releases/download/v<version>/shipboardai-<version>.tgz
olares-cli market upload shipboardai-<version>.tgz
olares-cli market upgrade shipboardai -s upload --version <version> --watch
```

- Tell the user before starting that Shipboard AI may be unavailable for a few minutes. Data (documents, records,
  conversations, settings) is kept, and so are app settings entered in Olares (for example the optional Langfuse host
  and keys); never change them.
- Prefer a quiet moment: answers being generated during the update are interrupted.
- Never downgrade (rule 7). If the installed version is newer than the latest release, report it and stop.
- Success: `shipboardai` shows `state: running` and the new `version`.

## 8. After any change

Run section 3 again and report the final table. Mention anything that is not `running`. Then section 9.

## 9. What Shipboard AI finishes by itself (tell the user)

Nothing here needs Lares or a command. After an install or update, Shipboard AI brings its search index to the
recommended setup in the background, at low priority (questions always go first):

- Documents registered with an older version get the document identification (kind, maker, product) that newer
  versions use to answer per product.
- If the app chose its models by itself (nobody changed them on 管理コンソール → システムの状態) and it searches with
  EmbeddingGemma, it re-embeds every document with Qwen3 Embedding 8B and switches search to it once everything is
  re-embedded. This can take several hours with many documents. Until then search keeps using EmbeddingGemma, so
  **EmbeddingGemma must stay running** (rule 2). After the switch, the user may stop EmbeddingGemma themselves; it is
  kept so an administrator can switch back on 管理コンソール → システムの状態.
- If an administrator chose the models on 管理コンソール → システムの状態, that choice is kept; the administrator can switch there.

Tell the user they can follow this on 管理コンソール → システムの状態 (models in use) and 管理コンソール → 処理状況 (processing).

## Notes

- Shipboard AI's own screen shows its version under 設定 (Settings). Lares cannot open it; ask the user to confirm it
  after an install or update.
