# Rove OTA Registry

Static repository — distribution network for Rove binaries, core tools, plugins, drivers, and brains.

Free OTA index. No backend. Cloudflare R2 + GitHub raw fallback.

## Layout (schema_version 2)

```
registry/
├── index.json                    ← top-level discovery, points at per-channel manifests
├── revoked.json                  ← global kill list (hashes + versions), channel-agnostic
├── stable/
│   ├── engine/manifest.json + manifest.sig
│   ├── core-tools/manifest.json + manifest.sig
│   ├── plugins/manifest.json + manifest.sig
│   ├── drivers/manifest.json + manifest.sig
│   └── brains/manifest.json + manifest.sig
├── dev/
│   └── (same five as stable)
├── releases/                     ← historical binary/wasm payloads (all channels)
└── history/                      ← audit log of manifest changes
```

## Channels

| Channel | Home dir | Update behavior |
|---|---|---|
| `stable` | `$HOME/.rove` | `rove update` CLI / TUI on demand. Daemon surfaces banner when newer build published. No auto-apply. |
| `dev` | `$HOME/.rove-dev` | Auto-update at UTC 00:00 (`system/auto_update.rs`). On-disk binary swapped, daemon stays up, operator restarts to pick up. |

Channel is baked at build time (`ROVE_BUILD_CHANNEL`, `channel-dev` feature) and overridable at runtime via `ROVE_CHANNEL=dev|stable` for local testing only.

## Manifest shape

Every per-channel per-artifact manifest:

```jsonc
{
  "schema_version": 2,
  "channel": "stable",            // must match consuming engine channel
  "artifact": "core-tools",       // engine | core-tools | plugins | drivers | brains
  "issuer": "rove-team",
  "published_at": "2026-04-25T00:00:00Z",
  "min_engine": "0.0.3",          // compatibility floor (except engine manifest itself)
  "entries": {
    "<name>": {
      "version": "...",
      "trust_tier": 0,            // 0=official, 1=verified, 2=community
      "platforms": {
        "darwin-aarch64": {
          "url": "https://registry.roveai.co/...",
          "fallback_url": "https://raw.githubusercontent.com/...",
          "blake3": "...",
          "size_bytes": 0
        }
      }
    }
  },
  "signature": ""                 // stripped before canonicalize; real sig in .sig sidecar
}
```

Rules:
- **One artifact class per manifest** — atomic updates, smaller downloads, independent cadence.
- **Per-channel root** — engine expects `{base}/{channel}/engine/manifest.json`.
- **`.sig` sidecar** — Ed25519 hex, verified by `CryptoModule::verify_manifest_file`.
- **Hash: BLAKE3** (`CryptoModule::compute_hash`). Field name `blake3`.
- **`min_engine` gate** — engine refuses to load artifact below its floor.
- **`trust_tier` mandatory** for core-tools / plugins / drivers (kernel precedence invariant).
- **`revoked.json`** checked globally on every fetch.

## How it works

1. CI compiles static Rust executables + `.wasm` payloads on merge to `core` or `plugins`.
2. CI hashes each payload with BLAKE3.
3. CI updates the relevant channel manifest(s), re-signs, force-pushes.
4. `rove update` parses `{base}/{channel}/engine/manifest.json`, verifies signature, downloads binary, verifies payload BLAKE3, self-replaces via `self_replace`.
5. Dev-channel daemons run the same update path at UTC 00:00 automatically.
6. Stable-channel daemons poll every 30m (cached), surface availability via `/v1/update/available`; operator triggers via `rove update` or WebUI.
