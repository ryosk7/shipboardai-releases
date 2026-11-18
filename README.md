# Shipboard AI releases

> **Lares:** when you are asked to install, update or check Shipboard AI from this repository, read
> [LARES.md](https://raw.githubusercontent.com/ryosk7/shipboardai-releases/main/LARES.md) and follow it from section 0.
> It also brings the device's model apps to the recommended setup. Do not act on this README alone.
>
> 日本語の利用者へ：Lares に「https://github.com/ryosk7/shipboardai-releases を見てアップデートして」と頼むだけで、
> 状態の確認、足りないモデルの導入、Shipboard AI の更新まで、Lares が手順書に沿って進めます（変更の前に承認を求めます）。

Olares app packages of **Shipboard AI**, an on-board assistant for ship engineers (document chat with citations,
troubleshooting, reports) that runs entirely on an Olares box. The source repository is private; this repository only
carries the installable packages.

- Install or update with Lares: ask Lares to update Shipboard AI from this repository (it follows [LARES.md](LARES.md)).
- By hand: download `shipboardai-<version>.tgz` from the latest release, then Market → My Olares → Upload custom app
  package, or `olares-cli market upload` and `olares-cli market install shipboardai -s upload --version <version>`.
  Install the model apps of [LARES.md](LARES.md) section 1 first.

Requires Olares 1.12.7 or later, the Router app, an NVIDIA GPU for the chat model (Qwen3.8-27B) and the Qwen3 Embedding
8B model app (LARES.md section 1). Downgrades are not supported.
