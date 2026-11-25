# VIctory fork (fxceless/victory-fork): progress record

The project: [fxceless/victory-fork](https://github.com/fxceless/victory-fork), a fork of
[VIctory](https://github.com/Marvisak/VIctory). It runs the user's own copy of an Xbox Series title natively on
Windows: the title's x86-64 code runs in-process, VIctory supplies the platform services, and the console's GPU code
is recompiled to DXIL for D3D12. Rust, Windows-only (`x86_64-pc-windows-msvc`). It is a separate project from the
Leonida work in `../PROGRESS.md`.

## Where the fork stands (from its README, 2026-11-25; measured on the fork author's PC, not by us)

Unlike Leonida, **the game already executes and renders under this fork**: boot, menus, the opening cutscene, saves,
and the first gameplay scene. Not playable yet, for two reasons:

1. **Everything in shadow is black.** The title's bounce light and its lamp and sun shadows are ray traced in its own
   compute shaders. The fork emulates the hardware ray/box and ray/triangle instruction, but the BVH structure
   builder stays off in real runs (it runs all or none, and some of its pipelines are refused or don't translate),
   and 5 of the 64 trace programs still don't translate. See the fork's `docs/fork-ray-tracing.md`.
2. **2 to 4 fps in the world** (240–450 ms per frame).

**The next blocker to work on is (1): the builder pipelines that hold the ray-tracing builder off.** Which pipelines
they are, and why, is only visible on a machine with the game and a real GPU.

## Session 2 (2026-11-25): build and tests, fork `main` @ `31714de`

Environment: cloud container, 4 cores, 15 GB RAM, no GPU. Rust 1.98.1 (pinned) with the MSVC target, clang/lld 18 as
`clang-cl`/`lld-link`, xwin 0.10.0 (MSVC CRT + Windows SDK 10.0.26100; running xwin accepts Microsoft's licence for
those files), Ubuntu Wine 9.0.

| Check | Result |
|---|---|
| Linux-native tests of the portable crates (core, formats, shader, pipeline, game, xtask) | ✅ 217 passed, 0 failed. The other crates are `cfg(windows)` |
| Windows cross-build of the workspace, all targets | ✅ |
| CI's clippy gates (`-D warnings`, with and without `--all-features`) | ✅ both clean |
| Release `victory.exe` under Wine: `--help`, `inspect`, `config` | ✅ |
| Windows tests under Wine (`cargo test --workspace`) | ⚠️ 476 passed, 28 failed, 8 ignored. **None traced to the fork's code**; see below |

The 28 failures, by cause (all environmental):

| Tests | Cause |
|---|---|
| 23 in `victory-shader` (`valu_tests`, `intersect_tests`, `waterfall_tests`) | Need a D3D12 device with shader model 6.6. Wine's D3D12 here has none ("no SM 6.6 device available"). These include the ray/node intersection tests, which matter for the shadow work: **run them on the real PC** |
| `xtask` `repository_passes_policy` | Spawns `git`; there is no Windows `git.exe` inside Wine. Passes in the Linux-native run |
| `victory-loader` `load_probe::loads_binds_and_calls_probe` | Spawns `cargo` (a Linux binary) from the Windows process |
| `victory-hle` `display::sdr_and_display_deferrals` | `0x80070578` ERROR_INVALID_WINDOW_HANDLE: no real window/display |
| `victory-hle` `net::only_loopback_is_reachable` | Wine's ws2_32 lacks `WSAConnectByList` |
| `victory-gpu` `host::sparse_copies_preserve_islands_and_observe_remaps` | Probably Wine: its `ReadProcessMemory` on the own process still reads a page made `PAGE_NOACCESS` (the test got the old bytes, `9`, where Windows refuses the read). Unverified until a Windows run |

Environment finding: **Ubuntu's Wine 9.0 crashes at startup (`free(): invalid pointer`) when `TMPDIR` is set**; it
calls `free()` on that variable's value. `unset TMPDIR` fixes it, so Ubuntu's Wine is enough (Session 1 switched to
WineHQ 11 for this crash). Seen in a backtrace inside `ntdll.so` `__wine_main`.

Change made: `patches/0001-Name-the-builder-stages-that-hold-the-ray-tracing-st.patch` (11 lines,
`crates/victory-gpu/src/pipeline.rs`). When `VICTORY_BVH_BUILDER=1` and the builder stays off, a diagnostics build now
logs every builder stage with no host pipeline and why: the shader hashes that don't translate, or the pipeline the
host refused. Before, only the count was logged. Clippy is clean with and without `--all-features`.

Small finding, not changed: `victory translate` reads two variables for the same setting, `VICTORY_HOST_WAVE`
(`apps/victory/src/main.rs`) and `VICTORY_HOST_MAX_WAVE` (`apps/victory/src/translate.rs`). Either works.

## Next steps

1. **On your PC** (Windows 11, the game in place, RTX 4070). Send back only the summarised lines named here, never
   game files:
   ```powershell
   git am path\to\0001-Name-the-builder-stages-that-hold-the-ray-tracing-st.patch
   # a. The shader tests on the real card (the 23 that cannot run here). Send the failed-test list.
   cargo test --workspace --locked
   # b. Offline: every shader through the translator AND host pipeline creation, as the 32-lane card sees it.
   #    The closing summary lists each failure reason with a count. Send that summary.
   $env:VICTORY_HOST_MAX_WAVE = '32'
   cargo run --release -p victory -- translate --pipelines
   # c. A run with the builder asked for. Send the lines containing "structure builder".
   cargo build --profile diag -p victory --features diagnostics
   $env:VICTORY_BVH_BUILDER = '1'; $env:VICTORY_BVH_QUARANTINE = '1'
   target\diag\victory.exe run <game dir>\<the title's exe> --execute 2>&1 | Select-String "structure builder"
   ```
   `VICTORY_BVH_QUARANTINE=1` keeps the builder's stores out of title memory, the safety step the fork's doc
   recommends for a first test.
2. Us: from those lines, fix the named builder stages (translation gaps or wave64 refusals), then the 5 shadow-mask
   trace programs, with unit tests built from the instruction patterns involved.
3. Not started: frame time.

## Reproduce this environment

```sh
rustup toolchain install 1.98.1 --profile minimal -c rustfmt -c clippy -t x86_64-pc-windows-msvc
cargo install xwin --locked
xwin --accept-license --sdk-version 10.0.26100 splat --use-winsysroot-style --preserve-ms-arch-notation --output ~/.local/share/xwin-sdk
# clang-cl, lld-link, llvm-lib: symlinks to /usr/lib/llvm-18/bin/{clang,lld-link,llvm-lib} in a dir on PATH
apt-get install -y wine64 xvfb xauth
. victory/winenv.sh                       # cross-build and Wine settings
cargo build --workspace --all-targets --locked --target x86_64-pc-windows-msvc
CARGO_TARGET_X86_64_PC_WINDOWS_MSVC_RUNNER=wine xvfb-run -a cargo test --workspace --locked --no-fail-fast --target x86_64-pc-windows-msvc
```
