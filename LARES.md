# Shipboard AI runbook for Lares

This file is written for **Lares** (the Olares AI assistant) to install, update and check **Shipboard AI** with
`olares-cli`. A person can read it too.

> 日本語の利用者へ：Lares にこのファイルの URL を渡し、「この手順書に沿って Shipboard AI の状態を確認して」と頼んでください。
> インストールや更新は、Lares が確認結果と実行内容を示したあとに、あなたが承認してから行われます。

## 0. Rules (read first)

1. **Report in the user's language** (Japanese for this vessel). Show the exact command before running anything that
   changes the device, and wait for the user's approval. Read-only commands (`list`, `get`, `curl` to GitHub) need no approval.
2. **Never** uninstall Shipboard AI or any other app, never delete uploaded charts, never stop apps you were not asked
   to stop, never change GPU / accelerator settings, never edit app env values. If a step seems to need one of these,
   stop and ask.
3. **Stop at the first failure.** Do not retry with different flags or another source. Report the command, its output
   and the state from `olares-cli market list --mine`.
4. **Idempotent**: if a step's goal is already met (app running, version already installed), skip it and say so.
5. Use only the package source in section 2. Do not download Shipboard AI from anywhere else.
6. Installing apps and uploading custom packages needs an **Olares administrator** and Lares' **write** (or full)
   permission. If a command is denied, stop and tell the user which permission is missing.

## 1. What Shipboard AI needs

| Olares app id | Title in Market | Required | Role |
|---|---|---|---|
| `router` | Router | yes | AI gateway; Shipboard AI calls every model through it. Shipboard AI cannot install without it. |
| `llamacppqwen3827bggufv3` | Qwen3.8-27B (llama.cpp) | yes | Chat model: answers, checks, report reviews, troubleshooting. Needs an NVIDIA GPU with about 23 GiB VRAM. |
| `llamacppllmbasev3` cloned as "Qwen3 Embedding 8B (llama.cpp)" (section 4) | llama.cpp Engine Base, Qwen3 Embedding 8B Q4_K_M on the CPU | yes (or EmbeddingGemma) | Embedding model for document search. Router lists it as `Olares/Qwen/Qwen3-Embedding-8B-GGUF:Q4_K_M`. Runs on the CPU (about 1 s per question, ~6 GiB RAM) so questions never switch the GPU away from Qwen3.8. |
| `embeddinggemmav3` | EmbeddingGemma | alternative | Lighter embedding model on the Intel iGPU; only if the user asks for it. One embedding model is enough. |
| `bgererankerv2m3` | BGE-Reranker-v2-m3 | recommended | Reranks search results (Intel iGPU). |
| `dotsocr` | dots.ocr | recommended | Reads scanned pages when documents are registered. |
| `levv1` | Lev System One | optional | Fast decision model; Shipboard AI records its decisions. Not required to answer. |
| `shipboardai` | Shipboard AI | — | The app itself, installed from an uploaded package (source `upload`). |

Olares must be **1.12.7 or later**.

## 2. Package source

- Releases: `https://github.com/ryosk7/shipboardai-releases/releases`
- Latest release (JSON): `https://api.github.com/repos/ryosk7/shipboardai-releases/releases/latest`
  - `tag_name` is `v<version>` (for example `v0.4.15`).
  - The asset `shipboardai-<version>.tgz` is the package; its `browser_download_url` is the download link.

## 3. Check (read-only) — always run this first

```sh
olares-cli market list --mine -o json
curl -s https://api.github.com/repos/ryosk7/shipboardai-releases/releases/latest
```

From the first output, read `state` and `version` of `router`, `llamacppqwen3827bggufv3`, the embedding model (a row
whose `title` contains `Qwen3 Embedding (llama.cpp)`, whose `name` is a clone such as `llamacppllmbasev314c33e`, or
else `embeddinggemmav3`; an older `ollamallmbasev3` row is no longer used and may stay stopped), `bgererankerv2m3`, `dotsocr`, `levv1` and `shipboardai`. From the
second, read the latest version (`tag_name` without the leading `v`).

Report a table like this, then propose the next section:

| App | Installed | State | Note |
|---|---|---|---|
| Router | 1.1.3 | running | ok |
| Qwen3.8-27B | 1.0.21 | running | ok |
| Qwen3 Embedding 8B (llama.cpp) | 1.2.40 | running | ok |
| Shipboard AI | 0.4.15 (latest 0.4.15) | running | up to date |

Decide:
- A required app is missing → section 4.
- A required app is `stopped` → section 5.
- Shipboard AI is missing → section 6.
- Shipboard AI is older than the latest release → section 7.
- Everything required is running and Shipboard AI is the latest → report "準備完了" and stop.
- Any state ending in `Failed` (for example `installFailed`, `upgradeFailed`) → stop and report; do not try to fix it.

## 4. Install a missing model app (needs approval)

Install only the required apps from section 1 that are missing, in this order: `router`, then
`llamacppqwen3827bggufv3`, then the embedding model.

```sh
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
```

- The Qwen model downloads tens of GB; a long wait is normal.
- Recommended and optional apps are installed only when the user asks.
- If the install asks for a GPU binding or compute mode you cannot answer from this file, stop and ask the user to
  install that app from Market in the browser instead.

## 5. Resume a stopped app (needs approval)

```sh
olares-cli market resume <app id> --watch
```

Only resume apps listed in section 1. If the resume needs a `--compute-binding`, stop and ask the user to resume it from
Market → My Olares in the browser.

## 6. Install Shipboard AI (needs approval)

Preconditions: `router`, `llamacppqwen3827bggufv3` and an embedding model are `running` (section 3).

```sh
curl -fL -o shipboardai-<version>.tgz \
  https://github.com/ryosk7/shipboardai-releases/releases/download/v<version>/shipboardai-<version>.tgz
olares-cli market upload shipboardai-<version>.tgz
olares-cli market install shipboardai -s upload --version <version> --watch
```

Success: `shipboardai` shows `state: running` and `version: <version>` with `source: upload`. Then tell the user to open
Shipboard AI from the Launchpad; the administrator finishes the setup in the app (vessel, crew, AI connection check).

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
- Never downgrade. If the installed version is newer than the latest release, report it and stop.
- Success: `shipboardai` shows `state: running` and the new `version`.

## 8. After any change

Run section 3 again and report the final table. Mention anything that is not `running`.

## Notes

- Shipboard AI's own screen shows its version under 設定 (Settings). Lares cannot open it; ask the user to confirm it
  after an install or update.
