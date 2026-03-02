# Zed Build Instructions

## Prerequisites

- macOS with Xcode and command line tools installed
- Rust toolchain (cargo)

## Build (Release)

From the `zed/` directory:

```bash
cargo build --release
```

The binary is output to `target/release/zed`.

Build takes ~6-7 minutes on a clean release build.

## Run

```bash
open target/release/zed
```

## Troubleshooting

### Stale path artifacts after moving the repo

If the repo has been moved from a different location, the build may fail with:

```
Unable to generate bindings: ParseCannotOpenFile { crate_name: "scene", src_path: "/old/path/crates/gpui/src/scene.rs" }
```

This happens because `gpui` embeds its directory path at compile time via `env!("CARGO_MANIFEST_DIR")`. Stale cached artifacts retain the old path.

**Fix:** Clean the affected crates and rebuild:

```bash
cargo clean -p gpui -p gpui_macos --release
cargo build --release
```
