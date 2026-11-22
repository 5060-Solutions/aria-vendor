# vendor — patched third-party crates for Aria

Three `-sys` crates. Two are patched so whisper.cpp and llama.cpp can live in
the same binary; `sherpa-rs-sys` is patched to build under Cargo 1.100's
build-dir layout. `aria-ai-core` references them via `[patch.crates-io]`.

## Why

Both crates vendor their own copy of ggml. Linking both fails:

```
ld.lld: error: duplicate symbol: ggml_backend_buffer_set_usage
```

On iOS it is worse — a Rust `staticlib` is only an archive, so the build
succeeds carrying two definitions and the clash is deferred to Xcode's final
link, where one copy is silently discarded. The two copies are different ggml
versions, so letting one win means one library calling the other's ggml across
mismatched struct layouts.

Upstream anticipates this: whisper.cpp has `WHISPER_USE_SYSTEM_GGML` and
llama.cpp has `LLAMA_USE_SYSTEM_GGML`. Neither Rust wrapper exposes the option.

## The patches

Each is marked `PATCH (aria)` in the source.

**llama-cpp-sys-2** — after its CMake build, publish where ggml's CMake package
was installed:

```rust
println!("cargo:ggml_dir={}", build_dir.join("lib/cmake/ggml").display());
```

Because the crate declares `links = "llama"`, cargo turns that into
`DEP_LLAMA_GGML_DIR` for any crate that depends on it.

**whisper-rs-sys** —

1. A new optional `system-ggml` feature adds a *normal* dependency on
   `llama-cpp-sys-2`. It is a normal dependency, not a build-dependency, because
   cargo only exports a `links` crate's metadata to crates that depend on it
   normally; it also forces the build order. Nothing from llama is used at
   runtime.
2. When `DEP_LLAMA_GGML_DIR` is present, pass `WHISPER_USE_SYSTEM_GGML=ON` and
   `ggml_DIR`, and stop emitting `-lggml*` — llama already links those.
3. Fix a genuine upstream bug: `build.rs` used `cfg!(target_os = "macos")` to
   decide whether to link `ggml-blas`, but inside a build script `cfg!` describes
   the **host**. Cross-compiling from a Mac to Android asked for a library CMake
   correctly never builds there. Now reads `CARGO_CFG_TARGET_OS`.

Both build scripts also carry `#![allow(warnings)]`: crates from the registry
compile with `--cap-lints allow`, but a `[patch]` path dependency does not, so
our `-Dwarnings` would otherwise apply this project's lint policy to upstream
code. Our own crates are unaffected.

**sherpa-rs-sys** — its build script copied the prebuilt onnxruntime and
sherpa-onnx dylibs into `target/<profile>/deps`, finding that directory by
walking up from `OUT_DIR` to the first ancestor named after `$PROFILE`, and
panicked if the copy failed. Cargo 1.100's build-dir layout no longer creates
`target/<profile>/deps`, so every build with the `download-binaries` default
failed with `Failed to copy ... to .../target/debug/deps/libonnxruntime.1.17.1.dylib`.

1. The dylibs are now copied into `$OUT_DIR/lib`, which Cargo defines and which
   the script already emits as a `rustc-link-search` path. Because it is inside
   the target dir, Cargo puts it on the dynamic-library search path for
   `cargo test` / `cargo run`, so nothing depends on Cargo's private layout.
2. Copies next to final artifacts (`target/<profile>`, `examples`, `deps`) are
   kept for running binaries outside Cargo, but only into directories that
   exist, and a failure is a `cargo:warning` instead of a panic. Locating
   `target/<profile>` returns `None` instead of `unwrap()`ing when the
   ancestor walk finds nothing (custom profiles, a separate `build.build-dir`).

## Verified

With the patches, `aria-ai-core --features stt-whisper-with-llm` links on the
host, Android arm64 and iOS arm64, and the combined static library contains
**one** definition of `ggml_init` and `ggml_backend_buffer_set_usage` (it was
two).

## Upstreams

- `llama-cpp-sys-2` — https://github.com/utilityai/llama-cpp-rs
- `whisper-rs-sys` — https://codeberg.org/tazz4843/whisper-rs
- `sherpa-rs-sys` — https://github.com/thewh1teagle/sherpa-rs

Sources are the crates.io packages (which already embed the C++ trees) rather
than git checkouts, so no submodule fetch is needed. On a version bump, re-apply
the `PATCH (aria)` hunks and re-run the verification above.
