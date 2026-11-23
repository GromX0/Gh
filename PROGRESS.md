# Leonida → GTA VI on PC: progress record

Goal: run the user's own copy of GTA VI (Xbox Series package 1.0.2.1) on their PC through
[Leonida](https://github.com/ItsNotRexed/Leonida). Leonida is a static recompilation: the x86-64 executable runs
natively, the console's system software is reimplemented as shims, and the RDNA2 shaders are translated to SPIR-V for
Vulkan.

This file is the persistent record across sessions. Update it at the end of every session.

## Evidence levels (never conflate these)

| Level | Meaning | Status (2026-11-23) |
|---|---|---|
| L0 Build | Code compiles | ✅ upstream `main` and our integration branch both build here (Linux native and Windows cross-build) |
| L1 Synthetic tests | Unit/semantic/GPU tests without game files pass | ✅ mostly, details in the log below |
| L2 Real executable maps | `gta6rt --dry-run` / `--init` on the user's `game_scarlett_master.exe` | ❌ **not run by us**; upstream measured it once (`docs/runtime_imports.md`, RT-5) |
| L3 Boot progresses | `gta6rt --run` advances through the retail boot chain | ❌ **not run by us**; upstream frontier is stop 17 (docs PR #6982), and PRs mention stops 19–20 |
| L4 Rendering | First draw and present on screen | ❌ **nobody has reached this.** Upstream says "Nothing has reached a draw or a present yet" |
| L5 Playable | Story mode playable | ❌ no |

Passing tests at L0/L1 says nothing about L2+. Only a run against the real executable moves L2–L5.

## The critical path to the real game

1. **User's copy, readable on disk** (blocker, user only, see below).
2. `make_game.py --check` recognises the copy as version 1.0.2.1.
3. `--stages shaders,runtime` (~30 GB): translate 26,503 shaders and build `gta6rt`.
4. `gta6rt --dry-run` → `--init` → `--run` with `--trace` and `--trace-d3d12`, triaged with `tools/boot_trace.py`
   (runbook: `docs/boot.md`).
5. Walk the retail boot chain (`docs/retail_boot_chain.md`): each stop is the first failure with every earlier fix
   in place. **`main` alone stops early**: most fixes for stops 3–20 are still unmerged PRs. That is why we keep an
   integration branch (below).
6. After the boot chain: the first draw and present (D3D12 → Vulkan), then the rest of the game.

## Hard blockers that only the user can resolve

1. **The game itself.** You need a copy of the Xbox Series package, version 1.0.2.1, with its files already
   readable as plain files on disk (the folder holding `MicrosoftGame.Config` and the `.rpf` archives), plus
   `game_scarlett_master.exe`. Upstream measured that file as 120,181,760 bytes, SHA-256
   `e95020b2fa4a721db3df59fdecf75621730ac559d8835252f6c3b658d5a08993`. We do not obtain the game, decrypt it, or
   work around any protection. Nothing of the game ever goes into this repository or this cloud session.
2. **A machine with a real GPU** to run it on: Windows 10/11 x64 (primary target) or Linux with Wine/Proton. This
   cloud session has no GPU (only Mesa llvmpipe) and must never receive game files.
3. **Hardware facts we need from you:** CPU vendor and model (an Intel CPU needs the Zen 2 instruction emulation from
   #7111, which is in our integration branch), GPU and driver version, OS, and free disk space (~30 GB to run, ~400 GB
   for a full asset export).

## Integration branch `boot-chain`

This is upstream `main` `042c5f37` plus our test fix, plus these open upstream PRs merged in boot-chain order. Every
conflict is resolved and described in its merge commit.

| Stop (`docs/retail_boot_chain.md`) | PR | Notes |
|---|---|---|
| 1, 2, 6, 10, 11 xmem | already on `main` (#6919) | #6709, #4531 and #6995 are superseded, not merged |
| 3 XErrorReport export | #6582 | |
| 4 TLS index 0 (`/GA`) | #4313 | loader TLS merge; `install_on_existing_threads` stays DLL-only |
| 4a GameInput ordinal 1 (Wine only) | #7216 | |
| 5 DStorageGetFactory | #4348 | |
| 7, 13, 15, 16, 17 slots 26/25/127, heaps at the title's address, render-target budget | #7240 (contains #6964, #7078, #7110, #7021, #7052) | #7235 skipped: the same hook mechanism with PC sizes, superseded by #7240's console tile sizes |
| 8, 9 slots 48/73, console list proxy | #7076 (contains #6935, #2490) | |
| 12 slot 133 | #7075 (contains #6953) | committed stand-in, not backed by the VA |
| 14 slot 70 | #7066 | the SRV write is a stand-in (descriptor left as is) |
| 19 slot 41 | #7163 | |
| 20 slot 61 | #7115 | |
| Intel CPUs: INSERTQ/EXTRQ, CLZERO, MONITORX/MWAITX, CLWB | #7111 (contains #7013) | |
| Diagnostics: null-call caller | #7189 | |

## Session log

### Session 1 — 2026-11-23

Environment: cloud container, Ubuntu 24.04, 4 cores, 15 GB RAM, no GPU (Mesa 25.2.8 llvmpipe), clang 18/19,
LLVM 19 (`llvm-mc` with amdgcn), Python 3.13, SPIRV-Tools v2025.1 (Ubuntu), Wine 9.0, xwin 0.10.0 (MSVC CRT +
Windows SDK 10.0.26100; running xwin accepts Microsoft's licence for those files).

Upstream `main` @ `042c5f37`, baseline results:

| Suite | Result |
|---|---|
| Linux runtime build (Vulkan headers 1.4.363, as CI) | ✅ 134/134 targets |
| Linux runtime ctest | ✅ 60/60 passed (2 skipped by design: game shader corpus, validation layer) |
| GPU tests on llvmpipe | ⚠️ 164 passed, 75 skipped, 1 failed: `ms_wave_info_s3` (wave64 mesh shader without emulation on 8-wide llvmpipe; draws nothing). It matches the llvmpipe mesh defects in `docs/testing.md`, and this Mesa is older than the versions measured there. Not a real-GPU result |
| Translator semantic suite (444 files, 5,902 tests) | ❌→✅ 4 files failed on `main`. **Fixed 2** (our commit `43e3cffc`): `test_image_atomics_guard` (47 failures, a test matcher written before #4473's `%active` caching) and `test_image_store_d16` (stale refusal of cube d16 after #6805; the output passes `spirv-val`). 2 left are tool-version only: `test_fma_khr` (this SPIRV-Tools lacks `FMAKHR`) and `test_int32_to_f64` (`spirv-opt` segfaults) |
| pytest set (CI `translator-tests` selection) | ⚠️ 1,821 passed, 9 skipped, 3 failed in asset/report tools off the boot path (`test_tex_export` ×2, `test_user_sgpr_census` folding-volume case). Not investigated |
| Windows cross-build (clang-cl 19 + xwin) | ✅ 384/384 (the first try failed on our setup: it picked up the distro's Vulkan 1.3.275 headers instead of the pinned 1.4.363) |
| Windows tests under Wine | Ubuntu's Wine 9.0 crashes at startup in this container (`free(): invalid pointer`), so we used WineHQ stable 11.0 (CI's other flavour). ⚠️ **219/240 passed, 21 failed.** All six CI-required tests pass (loader, pe_loader, loader_tls, unimplemented_policy, run_tiny_game, run_tiny_game_exit_code). The 21 failures trace to this headless environment: no DXGI/D3D12 under Wine (device_proxy: "no DXGI factory"), no window under Xvfb (d3d12_present, dxgi_swapchain), and fixture paths that expect a multi-config `RelWithDebInfo\` layout. This list is the **baseline**: the integration branch must add no new failures |

Integration branch `boot-chain` @ `3477a271` (bundle in `leonida/`):

| Suite | Result |
|---|---|
| Windows cross-build | ✅ 571/571, after one fix of ours (`xs_proxy_mock_test`: #7075 and #7240 each added a slot-27 mock; merged into one, and a `constexpr` enum cast clang rejects made `const`) |
| Windows tests under Wine 11 | 232/248 passed, 16 failed. Compared with the baseline: **no regression traced to the merges.** `d3d12_residency` and `d3d12_heap_properties` fail the same way on plain `main` when run alone (Wine's host D3D12 differs from run to run). The two new failing tests come from the merged PRs: `run_tiny_tls_ga` (#4313) passes its real check (`tls_index=0 main=42 thread=42 OK` under `--run`) and fails only the negative case, because Wine doesn't pass a Unix `TMP`/`TEMP` through to the Windows process. `dstorage` (#4348) fails one stats-timing check that pauses a copy on a `PAGE_GUARD` page; probably Wine's guard-page handling, **unverified until a Windows run** |
| Linux build + ctest | ✅ 60/60 |
| `tests/test_cpu_feature_scan.py` (#7111) | ✅ 22/22. Under Wine, `gta6rt` also reported "this CPU lacks SSE4a CLZERO MONITORX; the game's uses are emulated" on this Intel host, so the detection runs |

### Next steps (in order)

1. **User, on their PC:** follow "First run on your PC" below and send back the outputs listed there.
2. Us: triage the trace with `tools/boot_trace.py`, find the first stop on your machine, and fix it (or merge the
   upstream PR that does).
3. Keep the bundle current. Upstream moves fast (dozens of RT-5 PRs a day): re-check open PRs for stops 17+ each
   session, and drop our merges as they land on `main`.
4. Not started: anything past stop 20, the first draw and present, shaders on a real GPU through the game.

### First run on your PC (what to run and send back)

Nothing here sends game data anywhere. Send back **only** the summaries named below, never game files, traces quoting
game strings, or shader dumps.

```powershell
# 1. Hardware facts
python -I tools/pc_checker.py --check --json          # from the Leonida checkout; send the JSON
# 2. Check your copy (reads sizes; add --full-check for SHA-256 of every file)
python -I make_game.py --check --game "D:\path\to\package"   # send the last ~20 lines
# 3. Build what is needed to run (about 30 GB), from the boot-chain checkout
python -I make_game.py --game "D:\path\to\package" --stages check,shaders,runtime
# 4. Dry run and init (no game code runs in --dry-run)
$env:LEONIDA_D3D12 = 'vulkan'
.\runtime\build\gta6rt.exe --dry-run "D:\path\to\game_scarlett_master.exe" --game-dir "D:\path\to\package" --overlay "$env:TEMP\leonida-overlay"
.\runtime\build\gta6rt.exe --init    "D:\path\to\game_scarlett_master.exe" --game-dir "D:\path\to\package" --overlay "$env:TEMP\leonida-overlay"
# 5. First real run, network OFF (the executable's ws2_32 imports bind to the host)
.\runtime\build\gta6rt.exe --run "D:\path\to\game_scarlett_master.exe" --game-dir "D:\path\to\package" `
    --overlay "$env:TEMP\leonida-overlay" --trace "$env:TEMP\boot.jsonl" --trace-d3d12 2>&1 |
    ForEach-Object { $_.ToString() } | Out-File -LiteralPath "$env:TEMP\d3d12.log" -Width 4096 -Encoding utf8
python -I tools\boot_trace.py "$env:TEMP\boot.jsonl" --d3d12 "$env:TEMP\d3d12.log"   # send this summary
# 6. The runtime tests on real Windows (settles the Wine-only questions above)
ctest --test-dir runtime\build -C RelWithDebInfo --output-on-failure   # send the failed-test list
```

### How to reproduce this environment (cloud session)

```sh
apt-get install -y libvulkan-dev mesa-vulkan-drivers vulkan-tools spirv-tools llvm-19 clang-19 lld-19 xvfb xauth
# WineHQ stable (key checked with scripts/check_apt_key.py against D43F640145369C51D786DDEA76F1A20FF987672F)
cargo install xwin --locked
xwin --accept-license --sdk-version 10.0.26100 splat --use-winsysroot-style --preserve-ms-arch-notation --output ~/.local/share/xwin-sdk
# clang-cl 19: symlink /usr/lib/llvm-19/bin/clang as clang-cl (plus lld-link, llvm-lib, llvm-rc, llvm-mt) in a dir on PATH
# Vulkan-Headers at 6802bb4733b63ed5efd3adb308a6c885ef180ea1 (vulkan-sdk-1.4.363.0), passed as VULKAN_SDK / -DVulkan_INCLUDE_DIR
```
