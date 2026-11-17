# Shipboard AI releases

Olares app packages of **Shipboard AI**, an on-board assistant for ship engineers (document chat with citations,
troubleshooting, reports) that runs entirely on an Olares box. The source repository is private; this repository only
carries the installable packages.

- Install or update with Lares: give Lares the URL of [LARES.md](LARES.md) and ask it to check Shipboard AI.
- By hand: download `shipboardai-<version>.tgz` from the latest release, then Market → My Olares → Upload custom app
  package, or `olares-cli market upload` and `olares-cli market install shipboardai -s upload --version <version>`.

Requires Olares 1.12.7 or later, the Router app, an NVIDIA GPU for the chat model (Qwen3.8-27B) and an embedding model.
