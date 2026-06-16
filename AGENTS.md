# sync-kobo-and-workstation - AI Coding Agent Instructions

Canonical instructions for AI coding agents working in this repository. Tool-specific entrypoints symlink to this file where supported:
- `CLAUDE.md`
- `.cursorrules`
- `.github/copilot-instructions.md`

## Build and Run

```shell
# Build
cargo build --release

# Run (requires a mounted Kobo at the default or specified path)
cargo run -- --documents-directories="$HOME/Documents"
cargo run -- --dry-run --documents-directories="$HOME/Documents"
cargo run -- --kobo-directory=/path/to/kobo --documents-directories="$HOME/Documents"

# Check (fast compile check without producing a binary)
cargo check

# Format
cargo fmt

# Lint
cargo clippy
```

There are no tests in this project currently.

## Architecture

Single-binary async Rust CLI (2024 edition, `#![forbid(unsafe_code)]`) that copies EPUB and PDF files from local document directories to a mounted Kobo e-reader volume.

All logic lives in `src/main.rs`. The runtime is Tokio. The design is a concurrent producer-consumer pipeline connected by bounded `mpsc` channels:

1. **`find_books`** -- walks source directories with `async-walkdir`, filters by file extension, sends discovered book paths into the `book_path` channel and emits `FoundSrcDocument` stats.
2. **`sync_books`** -- receives from the `book_path` channel, checks whether each book already exists at the destination, and spawns a Tokio task per copy via `copy_to_non_existant`. Emits `Copied` or `NotCopiedBecauseAlreadyExistedAtDest` stats.
3. **`collect_stats`** -- receives from the `stats` channel and prints a summary when all producers drop their senders.

CLI argument parsing uses `clap` derive. A `PartialArgs` struct captures optional CLI values, which `parse_args` resolves into a fully populated `Args` by applying Linux-specific defaults (udisks2 mount path, `~/Documents`).

## Known Quirks

- `EXTENSIONS_TO_SYNCHRONISE` contains `"epub"` (no dot) and `".pdf"` (with dot) -- an inconsistency that affects matching.
- The function `copy_to_non_existant` is intentionally misspelt in the current codebase.
- Defaults are Linux-specific; macOS or other OS usage requires explicit `--kobo-directory` and `--documents-directories` flags.
