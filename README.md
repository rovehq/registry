# Rove OTA Registry

This static repository is the beating heart of Rove's distribution network.

Rather than standing up an expensive, heavily-rate-limited backend API server, Rove leverages this repository as a completely free Over-The-Air indexing service.

## ⚙️ How It Works

1. **Continuous Integration**: When a developer merges code into the `core` or `plugins` repositories, a GitHub Action automatically triggers.
2. **Compilation**: The action compiles static Rust executables (Windows, macOS, Linux) and standalone `.wasm` plugin payloads.
3. **Hashing**: Each payload receives a definitive `BLAKE3` hash ensuring cryptographically secure updates.
4. **Push**: The CI updates `manifest.json` pointing to the exact Cloudflare R2 bucket URLs or GitHub raw source fallbacks, and force-pushes immediately to this repo.

When a user runs `rove update`, the daemon parses `manifest.json` and patches the executable directly in-place without manual intervention.
