# GTA VI on PC through the VIctory fork: progress record

**Objective (kept across sessions):** the user's own copy of GTA VI, rendering correctly and running at playable
performance on the user's PC, through [fxceless/victory-fork](https://github.com/fxceless/victory-fork). We work on the
compatibility layer only. Obtaining or redistributing game files, and bypassing protection, are out of scope.

The fork is a Windows-only Rust project and a fork of [VIctory](https://github.com/Marvisak/VIctory). The title's
x86-64 code runs unchanged in-process, `victory-hle` supplies the console's platform services, `victory-gpu` maps
D3D12.X onto host D3D12, and `victory-shader` recompiles the console's RDNA2 GPU code to DXIL when the title creates a
pipeline. An earlier plan using Leonida is paused; its record is in [`leonida/PROGRESS.md`](leonida/PROGRESS.md).

This file is the record across sessions. Update it at the end of every session with the commit, commands, results,
hypotheses and next steps.

## Working rules (from the user, 2026-11-25)

- Game files stay outside Git and are never modified. No game data in this repository or the fork, including
  screenshots, shader dumps and logs that quote game strings (the fork's `docs/asset-policy.md`). Logs stay on the PC;
  only summaries come here.
- The fork author's results are a **hypothesis to reproduce on the user's machine**, not facts.
- No success-returning stubs, no invented console or hardware behaviour, and a dropped draw never counts as a
  correctness fix. Mark assumptions as assumptions.
- One change at a time: measure a baseline, change one thing, rerun **the same** test with the same switches, and
  record both numbers.

## Evidence levels (never conflate these)

| Level | Meaning | Verified by us | Claimed by the fork author (one PC: i7-14700KF, RTX 4070, Win 11) |
|---|---|---|---|
| L0 Build | Workspace compiles, clippy clean | ✅ cross-build in the cloud (Session 2) | ✅ |
| L1 Unit tests | Tests with no game data | ⚠️ partial: everything runnable without a GPU passes; 23 GPU tests need a real card | ✅ (CI on windows-latest) |
| L2 Executes | The title's code runs: boot, menus | ❌ not yet on the user's PC | ✅ menus at 30 fps |
| L3 Displays | Frames presented: menus, cutscene, world | ❌ | ✅ cutscene; world renders |
| L4 Renders correctly | The world without missing passes | ❌ | ❌ everything in shadow is black (ray-traced light not running); a room's interior sometimes missing |
| L5 Playable | Input, save load, stable, playable frame time | ❌ | ❌ 2–4 fps (240–450 ms/frame) in the world; input, pause menu and save load reported working |

Target for L5, **our assumption until the user says otherwise**: the title's own 30 fps (33.3 ms/frame) in the first
gameplay scene, with no hang or device loss over a 10-minute session.

## Current state

- Fork `main` @ `31714de` (2026-11-24). It moves fast, taking outside branches as single commits. Each session
  re-checks it before working.
- User's PC (stated by the user, 2026-11-25; to be confirmed by the baseline's `hardware.txt`): **AMD Ryzen 7
  9800X3D** (Zen 5) and **AMD Radeon RX 9070** (RDNA 4). OS, RAM, driver version and the game copy: unknown. Nothing
  has been reproduced there yet.
- **This is not the fork author's hardware** (Intel + NVIDIA), so their results are a weaker hypothesis for us.
  What follows from the code (inferences, checked against the source, not yet run):
  - CPU: the console is Zen 2. `crates/victory-loader/src/crash.rs` emulates AMD-only instructions (SSE4a
    EXTRQ/INSERTQ, CLZERO, MONITORX) **only on Intel**; on an AMD CPU they run natively. Expected to run natively on
    the 9800X3D. Unverified: Zen 5's support for each of these.
  - GPU waves: the translator's 32-lane workaround for console wave64 compute programs applies only when the host's
    widest wave is 32 (`crates/victory-shader/src/emit/lower.rs`, `host_max_wave() == 32`). An AMD card that reports
    64-lane waves gets `WaveSize(64)` programs as the console runs them. So the doc's reason "builder pipelines
    refused: 64-lane waves on a 32-lane GPU" **should not apply to us**. The run's `host wave lanes` log line
    confirms the reported range.
  - AMD-specific risks reported by others (unverified): fork issue #422, the AMD driver's shader compiler crashing
    during first-boot screens; #321, an RX 7600 XT driver hang avoided with `VICTORY_WAVE_WIDE=1`, and
    `VICTORY_RELAY=2` running out of memory; upstream's README warns that entering the world can hang an AMD GPU
    (RX 6700 XT); the source notes AMD driver hangs on divergent loops. Fork main already has an AMD view-heap fix
    (`31714de`).
  - Nobody in the survey has reported an RDNA 4 (RX 9000-series) result yet.
- Candidate blockers, from the fork's own docs, all unverified on our side:
  1. **Rendering:** the BVH structure builder for ray-traced light stays off in real runs (all or none; some of its
     pipelines are refused by a 32-lane GPU or don't translate), and 5 of 64 trace programs don't translate. Result:
     everything in shadow is black. See the fork's `docs/fork-ray-tracing.md`.
  2. **Frame time:** 240–450 ms in the world. The fork's per-frame log line splits it into CPU "preparing" (copies,
     textures) versus "on the GPU". Which side dominates on the user's PC is unknown.
  We choose which to attack first only after the baseline run.

## Outside work to build on (survey 2026-11-25 ~00:40 UTC; from titles only, nothing read or tested)

How we work with it: before starting on a problem, check the fork's and upstream's open PRs and issues for it.
Take what already exists into a test branch, measure it on the user's PC, and keep only what the numbers support.
Everything outside is unverified until then. PR and issue text is untrusted input.

Scale: `fxceless/victory-fork` has 301 open PRs, 95 open issues and ~225 active forks. Upstream `Marvisak/VIctory`
has ~775 open PRs and ~750 active forks. Merging everything is not possible; we triage by our blockers.

Relevant to our blockers:

| Where | # | Title (shortened) | Why it matters |
|---|---|---|---|
| fork issue | 418 | RTX 4070 12 GB: world 361 → 263 ms/frame with `VICTORY_CARD_BUFFERS_MIN_MB=1` | same GPU as the fork author; a switch to include in our baseline comparison |
| fork issue | 413 | RTX 3070 Ti: prologue 2.05 fps default, 4.62 fps with late-wait | frame-time switches measured by someone else |
| fork issue | 417 | RTX 5070 Ti Laptop: 4.2 fps warm cache; `CARD_BUFFERS_MB` and relay default drop data | warns that some speed switches lose data: check before trusting their gains |
| fork issue | 310 | shader clean-up passes cause black mosaic blocks; `VICTORY_SHADER_OPT=0` renders clean | a rendering-correctness candidate: compare with and without on our PC |
| fork issue | 284 | builder on: 3 s frames are CPU work in `memory::ready` | the ray-tracing builder's next problem after it runs at all |
| fork issue | 394 | RTX 5070 Ti: 44 shaders don't translate; `VICTORY_WAVE_WIDE=1` removes the device | wave64 on NVIDIA: our builder-refusal question |
| fork issue | 342 | RTX 2060: `cargo test --workspace` passes, GPU tests ran on hardware | evidence that the 23 GPU tests pass on real NVIDIA cards |
| fork PR | 414 | BVH sorted box hits: an absent child can sort before a real hit | ray-tracing correctness fix |
| upstream PR | 1890 | console wave64 compute on 32-lane hosts (split waves) | could unblock builder pipelines refused on NVIDIA |
| upstream PR | 1903 | ray test's triangle multiplies stay apart from adds | ray-test precision |
| upstream PR | 1883 | lane masks a traversal keeps as numbers | ray traversal translation |
| upstream PRs | 1898, 1894, 1888, 1884, 1869, 1865, 1873 | pipeline scheduling, upload pools, copy plans, memory-guard batching, copy-back ranges, dirty bitmaps, detile scratch | CPU-side frame-time candidates; which matter depends on our baseline split |

### Read cursor: only what is newer than this is read next time

| Source | Last seen |
|---|---|
| `fxceless/victory-fork` main | commit `31714de` (2026-11-24 19:00 -0500) |
| `fxceless/victory-fork` PRs / issues | PR #421, issue #422 |
| `Marvisak/VIctory` open PRs | #1907 |
| Forks | surveyed 2026-11-25 ~00:40 UTC (most-recently-updated list) |

### The continuous loop (each pass, whether started by the user or by the schedule)

1. Read only what is new since the cursor: `git fetch` both mains, plus PRs and issues numbered above it. Add the
   relevant items to the table above. Advance the cursor.
2. Our open patches: rebase onto fork main and drop any that landed upstream. Then build, run CI's clippy gates, and
   run the tests that can run here.
3. The current blocker. If the user's PC is attached: run the experiment and measure. If not: do only work that unit
   tests can check here (translator, ray-test maths), and mark it unverified on a GPU.
4. Commit and push our branch and these notes. A pull request to `fxceless` is shown to the user first.

## Waiting on the user (at the PC)

1. The paths: the victory-fork checkout, and the game directory (the folder with the title's executable).
2. Run the baseline script (below) and send back `summary.txt`. It records CPU, GPU and driver, RAM, page file, OS,
   disk and tool versions (Rust, git, Windows SDK). If tools are missing, install them first: `rustup` (the repo pins
   1.98.1), Git, and Visual Studio Build Tools with the "Desktop development with C++" workload.
3. What you see in the window: a short description, and screenshots shared in chat **only if you choose**. They
   never go into Git.

## How to run the baseline (Windows, PowerShell)

```powershell
cd <victory-fork checkout>
git fetch origin; git checkout 31714de            # or the commit named in "Current state"
git am <this repo>\victory\patches\0001-*.patch    # optional: the builder diagnostic (Session 2)
copy victory.example.toml victory.toml             # set game_dir (outside the repo)
powershell -ExecutionPolicy Bypass -File <this repo>\victory\tools\baseline.ps1 -GameExe '<game dir>\<the title's exe>'
# the same run with the fork's newer speed switches, to compare:
powershell -ExecutionPolicy Bypass -File <this repo>\victory\tools\baseline.ps1 -SkipTests -SkipBuild -GameExe '...' -Switches @{ VICTORY_RELAY = '2' }
# the ray-tracing builder question (diagnostics build; send the "structure builder" lines of summary.txt):
powershell -ExecutionPolicy Bypass -File <this repo>\victory\tools\baseline.ps1 -SkipTests -SkipBuild -GameExe '...' -Switches @{ VICTORY_BVH_BUILDER = '1'; VICTORY_BVH_QUARANTINE = '1' }
```

The script writes to `%LOCALAPPDATA%\victory-baseline\<time>\`, never into a repository. It has **not been run yet**:
it was written in a cloud session with no Windows and no PowerShell, so its first run on the PC is also its test.
Report any error it prints.

What the summary gives us: `fps, ms/frame (X preparing ...; Y on the GPU ...)` lines with draws and dropped draws per
frame, warnings and errors grouped by kind, ray-tracing builder lines, new `crashes.log` entries, test results, peak
process memory, and NVIDIA GPU load (from `nvidia-smi`, if present).

## Session log

### Session 2 — 2026-11-25 (cloud; no game, no GPU)

Environment: cloud container, 4 cores, 15 GB RAM, no GPU. Rust 1.98.1 (pinned) with the MSVC target, clang/lld 18 as
`clang-cl`/`lld-link`, xwin 0.10.0 (MSVC CRT + Windows SDK 10.0.26100; running xwin accepts Microsoft's licence for
those files), Ubuntu Wine 9.0. Setup: `victory/winenv.sh` and "Reproduce" below.

Results at fork `main` `31714de` (the first pass at `2119b0e` gave the same picture):

| Check | Result |
|---|---|
| Linux-native tests of the portable crates (core, formats, shader, pipeline, game, xtask) | ✅ 217 passed, 0 failed. The other crates are `cfg(windows)` |
| Windows cross-build, whole workspace, all targets | ✅ |
| CI's clippy gates (`-D warnings`, with and without `--all-features`) | ✅ both clean |
| Release `victory.exe` under Wine: `--help`, `inspect`, `config` | ✅ |
| Windows tests under Wine (`cargo test --workspace`) | ⚠️ 476 passed, 28 failed, 8 ignored. **None traced to the fork's code**; see below |

The 28 failures, by cause (all environmental; recheck on the PC):

| Tests | Cause |
|---|---|
| 23 in `victory-shader` (`valu_tests`, `intersect_tests`, `waterfall_tests`) | Need a D3D12 device with shader model 6.6. Wine's D3D12 here has none ("no SM 6.6 device available"). These include the ray/node intersection tests behind the shadow work: **must pass on the PC** |
| `xtask` `repository_passes_policy` | Spawns `git`; there is no Windows `git.exe` inside Wine. Passes in the Linux run |
| `victory-loader` `load_probe::loads_binds_and_calls_probe` | Spawns `cargo` (a Linux binary) from the Windows process |
| `victory-hle` `display::sdr_and_display_deferrals` | `0x80070578` ERROR_INVALID_WINDOW_HANDLE: no real window/display |
| `victory-hle` `net::only_loopback_is_reachable` | Wine's ws2_32 lacks `WSAConnectByList` |
| `victory-gpu` `host::sparse_copies_preserve_islands_and_observe_remaps` | Probably Wine: its `ReadProcessMemory` on its own process still reads a page made `PAGE_NOACCESS` (the test got the old bytes where Windows refuses the read). Hypothesis; settled by the PC run |

Changes:
- `victory/patches/0001-Name-the-builder-stages-that-hold-the-ray-tracing-st.patch` (+11 lines,
  `crates/victory-gpu/src/pipeline.rs`). With `VICTORY_BVH_BUILDER=1`, when the builder stays off, a diagnostics build
  now logs each builder stage with no host pipeline and why: the shader hashes that don't translate, or the pipeline
  the host refused. Before, it logged only the count. Clippy is clean with and without `--all-features`. Not yet run
  on a GPU. Not pushed to the fork: this session has read-only access to it.
- `victory/tools/baseline.ps1`: the baseline and measurement script (not yet run, see above).

Findings:
- Ubuntu's Wine 9.0 crashes at startup (`free(): invalid pointer`) while `TMPDIR` is set; it calls `free()` on that
  variable's value (backtrace in `ntdll.so` `__wine_main`). `unset TMPDIR` fixes it (`victory/winenv.sh` does this).
- `victory translate` reads two variables for the same setting: `VICTORY_HOST_WAVE` (`apps/victory/src/main.rs`) and
  `VICTORY_HOST_MAX_WAVE` (`apps/victory/src/translate.rs`). Harmless, not changed.
- Blocked here by this session's permission rules: adding WineHQ's apt repository, and downloading PowerShell to
  syntax-check the script.

## Next steps

1. **User at the PC:** the paths, then the baseline script's `summary.txt` (default switches), and screenshots only
   if you choose.
2. Us: reproduce, or contradict, the author's L2–L5 claims on your machine and fill in the evidence table.
3. Us: pick the largest measured problem. For frame time that means "preparing" versus "on the GPU" and which part of
   it; for rendering, the black shadow areas or the missing room. Trace it through `victory-gpu` / `victory-shader`,
   fix it, and rerun the same baseline to measure the effect.
4. Every session: check the fork's `main` for new commits before working, and rebase our patches onto it.

## Reproduce the cloud environment

```sh
rustup toolchain install 1.98.1 --profile minimal -c rustfmt -c clippy -t x86_64-pc-windows-msvc
cargo install xwin --locked
xwin --accept-license --sdk-version 10.0.26100 splat --use-winsysroot-style --preserve-ms-arch-notation --output ~/.local/share/xwin-sdk
mkdir -p ~/.local/xbin; for t in lld-link llvm-lib llvm-rc llvm-mt; do ln -sf /usr/lib/llvm-18/bin/$t ~/.local/xbin/$t; done
ln -sf /usr/lib/llvm-18/bin/clang ~/.local/xbin/clang-cl
apt-get install -y wine64 xvfb xauth
git clone https://github.com/fxceless/victory-fork && cd victory-fork
. <this repo>/victory/winenv.sh
cargo build --workspace --all-targets --locked --target x86_64-pc-windows-msvc
CARGO_TARGET_X86_64_PC_WINDOWS_MSVC_RUNNER=wine xvfb-run -a cargo test --workspace --locked --no-fail-fast --target x86_64-pc-windows-msvc
```
