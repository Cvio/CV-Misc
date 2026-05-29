# Handy — Windows Build Log

A running record of getting [cjpais/Handy](https://github.com/cjpais/Handy) (offline speech-to-text, Tauri/Rust/React) building from source on Windows. Living document — appended as we go.

**Goal:** Get Handy running locally on the 4070 laptop as a starting point toward a standalone, offline translator (transcribe → translate via local LLM, no internet at runtime).

**Current state:** Build succeeds, `bun tauri dev` launches. Vulkan/GPU-Whisper dropped in favor of CPU Whisper + DirectML Parakeet.

---

## TL;DR — clean Windows build from scratch (the working recipe)

For a fresh machine, in order. This is the whole gauntlet condensed:

1. **Prereqs already needed:** Rust (+MSVC target), Bun, MSVC C++ Build Tools.
2. `winget install LLVM.LLVM` — provides libclang (whisper.cpp's bindgen needs it).
3. `winget install Kitware.CMake` — whisper.cpp's build system. Verify `cmake --version` in a fresh shell; add `C:\Program Files\CMake\bin` to PATH if missing.
4. In `src-tauri/Cargo.toml`, change the Windows `transcribe-rs` features from `["whisper-vulkan", "ort-directml"]` to `["ort-directml"]`. (Avoids the fragile Vulkan shader build; keeps GPU Parakeet via DirectML.) **This removes the need for the Vulkan SDK entirely.**
5. Add to `.cargo/config.toml` so libclang persists across shells:
   ```toml
   [env]
   LIBCLANG_PATH = "C:\\Program Files\\LLVM\\bin"
   ```
6. `bun install`, then `bun tauri dev`.

> Note: if you *keep* `whisper-vulkan` (step 4 unchanged), you additionally need `winget install KhronosGroup.VulkanSDK` and must debug the shader-gen compile under your toolchain. Not recommended unless you specifically want GPU-accelerated Whisper.

---

## Status legend

- **Confirmed** — verified working on this machine
- **Issued** — instructions given, result not yet confirmed
- **Pending** — not started yet

---

## Machine / environment

| Thing | Value |
|---|---|
| Machine | Windows laptop, RTX 4070 Laptop GPU (8GB VRAM), 32GB RAM |
| Shell | Git Bash (primary) |
| Rust | rustc 1.94.1 / cargo 1.94.1 — **Confirmed present** |
| Bun | 1.3.14 — **Confirmed installed** |
| MSVC | VS 2026 (v18) Build Tools, MSVC 14.50.35717 (seen in build paths) |
| LLVM / libclang | installed via winget, default path `C:\Program Files\LLVM\bin` — **Confirmed** |
| Vulkan SDK | `KhronosGroup.VulkanSDK` via winget — **Confirmed** (build got past the VULKAN_SDK check) |
| CMake | `Kitware.CMake` via winget — **Confirmed** (configure + build steps now run) |
| Toolchain note | VS 2026 (v18) is bleeding-edge; the whisper.cpp Vulkan compile failed under it |

---

## What the tools are (plain version)

- **Rust** — the language Handy's backend is written in.
- **Cargo** — Rust's build + package manager (downloads Rust libraries from crates.io, compiles everything).
- **Bun** — the equivalent for the JavaScript frontend (installs JS libraries, runs scripts).
- **Tauri** — framework that staples the Rust backend to a web-tech UI and packages it as one native desktop app. It's why the end result is a single installer, not a browser tab.

None of these four are needed on the *target* machine. They're build-time only. The shipped app runs on its own.

---

## Internet requirements (settled)

- **Build time:** needs internet **once**. `bun install` pulls JS libs from npm; `cargo` pulls Rust libs from crates.io. Both cache to disk (project folder + `~/.cargo`), so rebuilds work offline once the cache is warm.
- **First run:** transcription models download from `blob.handy.computer`. Can be pre-staged manually (see README "Manual Model Installation") so an offline machine never phones home.
- **Steady-state runtime:** fully offline **if** (a) the model is pre-staged and (b) translation goes through local Ollama (the "Custom" post-process provider, default `http://localhost:11434/v1`) instead of a cloud API. Cloud providers are an option, not a requirement.

---

## Setup steps

### 1. Verify Rust — Confirmed
```bash
rustc --version   # rustc 1.94.1
cargo --version   # cargo 1.94.1
```

### 2. Install Bun — Confirmed
```bash
powershell -c "irm bun.sh/install.ps1 | iex"
# reopen Git Bash, then:
bun --version     # 1.3.14
```

### 3. Install JS dependencies — Confirmed
```bash
bun install       # completed cleanly
```

### 4. First `bun tauri dev` — failed (see Error 1)

---

## Errors and fixes (chronological)

### Error 1 — `libclang` not found — RESOLVED

**Symptom:**
```
thread 'main' panicked at bindgen-0.72.1\lib.rs:
Unable to find libclang: "couldn't find any valid shared libraries matching:
['clang.dll', 'libclang.dll'], set the `LIBCLANG_PATH` environment variable ..."
```

**Cause:** `whisper-rs` compiles whisper.cpp from C source and uses `bindgen` to auto-generate the Rust↔C glue by parsing C headers. The parser is `libclang`, shipped with LLVM. LLVM wasn't installed.

**Note:** Not avoidable by choosing Parakeet over Whisper — both transcription engines compile into the app regardless of which is used at runtime, so the whisper.cpp build must succeed either way.

**Fix:**
```bash
winget install LLVM.LLVM          # installs to C:\Program Files\LLVM
```
Then make `LIBCLANG_PATH` persistent (an `export` dies with the shell session). Chosen method: in-repo via `.cargo/config.toml`, because it's version-controlled, visible, and survives shell restarts.

```toml
# .cargo/config.toml
[env]
LIBCLANG_PATH = "C:\\Program Files\\LLVM\\bin"
```

**Why not the app entrypoint?** `LIBCLANG_PATH` is a *build-time* variable — libclang is only used while compiling. The running app never touches it, so putting it in `main.rs` or a startup script would do nothing. Build deps belong in the build environment.

**Why not the Nix flake?** Handy ships a Nix flake (the "declare the whole toolchain reproducibly" answer), but it only targets `x86_64-linux` and `aarch64-linux`. No Windows support, so it doesn't help here.

### Error 2 — `VULKAN_SDK` not set — RESOLVED

**Symptom:**
```
thread 'main' panicked at whisper-rs-sys-0.15.0\build.rs:226:
Please install Vulkan SDK and ensure that VULKAN_SDK env variable is set
```

**Cause:** whisper.cpp is compiled with its Vulkan GPU-acceleration backend on, which needs the Vulkan SDK + a `VULKAN_SDK` env var.

**Fix (issued, order matters):**
1. Persist `LIBCLANG_PATH` in `.cargo/config.toml` *first* (so the upcoming shell restart doesn't undo Error 1's fix).
2. `winget install KhronosGroup.VulkanSDK` — installer sets `VULKAN_SDK` system-wide and adds its Bin to PATH automatically.
3. **Close Git Bash fully and open a fresh window** — running shells don't see the new system variable until restarted. `cd` back into `Handy`.
4. Re-run `bun tauri dev`.

**Status:** Resolved — build advanced past the `VULKAN_SDK` panic and began invoking cmake with `-DGGML_VULKAN=ON`. MSVC toolchain also auto-detected ("Visual Studio 18 2026" generator).

### Error 3 — `cmake` not installed — RESOLVED

**Symptom:**
```
running: "cmake" ... "-G" "Visual Studio 18 2026" ... "-DGGML_VULKAN=ON" ...
thread 'main' panicked at cmake-0.1.57\src\lib.rs:1132:
failed to execute command: program not found
is `cmake` not installed?
```

**Cause:** whisper.cpp uses CMake as its build system. The Rust `cmake` crate shells out to the `cmake` executable, which isn't on PATH. (Vulkan + MSVC were both found — Vulkan via the env var, MSVC via the auto-detected VS 2026 generator — so this is purely the missing build orchestrator.)

**Fix (issued):**
1. `winget install Kitware.CMake`
2. Restart Git Bash (PATH change), `cd` into `Handy`, verify `cmake --version`. CMake's installer is notorious for not adding itself to PATH by default — if "command not found," re-run the installer choosing "Add CMake to the system PATH," or add `C:\Program Files\CMake\bin` to PATH via the env-vars GUI.
3. Re-run `bun tauri dev`.

**Status:** Resolved — CMake installed and on PATH. Configure step completed (the `CMP0128/CMP0156/CMP0200` "CMake Warning (dev)" lines are harmless policy warnings aimed at project devs, not errors). Build proceeded to the actual compile step, which then failed — see Error 4.

### Error 4 — whisper.cpp Vulkan compile fails — RESOLVED

**Symptom:**
```
running: "cmake" "--build" ... "--target" "install" "--config" "Release" "--parallel" "20"
thread 'main' panicked at cmake-0.1.57\src\lib.rs:1132:
command did not execute successfully, got: exit code: 1
```
The panic is just the wrapper reporting the build sub-command failed; the real compiler/shader error scrolled off above it and wasn't captured.

**Cause (most likely):** the Vulkan backend's GLSL→SPIR-V shader-compilation step is the most fragile part of the whisper.cpp build, and this is a brand-new VS 2026 toolchain. Force-enabled by a feature flag (see fix).

**Strategic fix — drop the Vulkan feature instead of debugging it.** GPU Whisper is the README's crash-prone path, the plan is to run Parakeet anyway, and the shader step is the likeliest culprit. In `src-tauri/Cargo.toml`:

```toml
# BEFORE
[target.'cfg(windows)'.dependencies]
transcribe-rs = { version = "0.3.3", features = ["whisper-vulkan", "ort-directml"] }

# AFTER
[target.'cfg(windows)'.dependencies]
transcribe-rs = { version = "0.3.3", features = ["ort-directml"] }
```

Why this works: the base `[dependencies]` block already requests `whisper-cpp` (CPU Whisper) + `onnx`, and Cargo unions features across blocks. Dropping `whisper-vulkan` leaves CPU-only Whisper (builds without shaders or Vulkan) plus `ort-directml` — the ONNX-runtime **DirectML** backend, i.e. GPU-accelerated **Parakeet** on the 4070 via DirectX. Net: lose GPU Whisper (unwanted), keep GPU Parakeet (the actual engine), shader mess gone.

Re-run `bun tauri dev` after the edit (feature change auto-triggers rebuild). If `whisper-rs-sys` is stale from the failed run: `cd src-tauri && cargo clean -p whisper-rs-sys` then re-run.

**Caveat:** actual compile error wasn't captured, so this isn't guaranteed to be the *only* problem — there's a small chance of a CPU-side whisper.cpp issue under VS 2026. If an error survives, capture it: `bun tauri dev 2>&1 | tee build.log` then `grep -i error build.log`.

**Status:** RESOLVED — dropping `whisper-vulkan` cleared it. The build compiled through and `bun tauri dev` launched successfully. Confirms the Vulkan shader-gen step was the (only) blocker; the CPU whisper.cpp + DirectML path builds clean under VS 2026.

---

## Key learnings

- **`export` is session-only.** Anything you'll need across shell restarts goes in `.cargo/config.toml [env]` (build vars), `setx` (user env), or the installer sets it system-wide. Relevant because installing Vulkan forces a shell restart, which wipes any `export`.
- **Build-time vs runtime is the core distinction.** Most of these errors are missing build tools, not broken code or runtime problems. The shipped app needs none of them.
- **Both engines always compile in.** Runtime model choice (Whisper vs Parakeet) doesn't change what gets built.
- **Whisper-on-GPU is the crash-prone path** per the README (config-dependent crashes on some Windows/Linux setups). Mitigation: run **Parakeet V3** at runtime — CPU/ONNX, sidesteps the Vulkan path entirely, and does automatic language detection (useful when input language varies).
- **Machine-specific paths don't belong in the upstream repo.** The `.cargo/config.toml` LIBCLANG entry is fine in a personal fork but would break Mac/Linux builds if pushed upstream.

---

## Translation wiring (Ollama post-process) — WORKING END TO END

Full pipeline confirmed: speak English → Parakeet transcribes → qwen3:8b translates → Spanish pastes into the focused field. Fully local, no internet.

### Networking: WSL2 Ollama ↔ Windows Handy

- Ollama runs inside **WSL2 (Ubuntu)**, bound to `127.0.0.1:11434` *inside* WSL.
- Handy is a **Windows** app. It reaches Ollama via **WSL2 localhost forwarding**: Windows `localhost:11434` transparently forwards into WSL — **confirmed working**, no URL surgery needed.
- The WSL IP route (`172.31.61.110:11434`) does **not** work (Ollama binds localhost, not 0.0.0.0). Use localhost, not the WSL IP.
- **Operational catch:** forwarding only works while the Ollama server is running *and* WSL is alive. If WSL shuts down, Ollama goes with it and Handy's translation calls fail. Autostart-on-boot is a TODO for standalone use.

### `ollama serve` vs `ollama run`

- `ollama serve` = the background **API server** on port 11434 (what Handy needs).
- `ollama run <model>` = interactive terminal chat; it also requires/starts the server.
- On Linux/WSL the server typically **auto-starts on install**, so the API is just there — you don't need a terminal open. `ollama run` is not required for Handy to work. (Verify with `Invoke-RestMethod http://localhost:11434/api/tags` from Windows.)

### Model choice

- **qwen3:8b** — chosen. ~5GB, fits 8GB VRAM, fast, strong multilingual (esp. Chinese↔English + major European). Text-only, no vision-encoder spill.
- Rejected: `qwen3.5` (vision encoder spills ~28% to CPU, slower, no benefit for text translation), `qwen-tech` (technical fine-tune), `qwen3-coder-next` (51GB, coder, far too big).
- **Input language is bound by the transcription model** (Parakeet = 25 European languages). **Output language is bound by the LLM** (Qwen handles CJK, Arabic, etc.). So "speak English → get Chinese" works; "speak Arabic → anything" does not, because Parakeet can't hear Arabic. For non-European *input*, switch transcription to Whisper.

### Reasoning-model suppression (the qwen3 "thinking" gotcha)

- Qwen3 is a reasoning model; left alone it emits `<think>` blocks that would get pasted along with the translation.
- `/no_think` in the prompt: **did not work** (model thought anyway).
- `/set nothink` interactively: works, but only in a terminal session — Handy uses the API, can't send it.
- API flag `"think": false`: works (tested via `/api/chat`).
- **But none of this is the user's job:** Handy already sends `reasoning_effort: "none"` for the Custom provider on every call (see `actions.rs` ~line 132). Suppression is automatic. The prompt needs no `/no_think` line.

### Exact UI path in Handy (v0.8.3)

1. Settings via **tray icon right-click → Settings**, or **Ctrl+,**.
2. **Advanced** tab → APP section → turn on **Experimental Features** (post-processing is gated behind it).
3. This reveals a new **Post Process** tab in the sidebar.
4. Post Process tab → **Provider = Custom** → Base URL auto-fills `http://localhost:11434/v1` (leave it).
5. **API Key: leave blank** — Ollama doesn't validate it.
6. **Model:** click the refresh icon to query Ollama, then select **qwen3:8b**.
7. **Create New Prompt** → Label "Translate to Spanish", Instructions:
   ```
   Translate the text below into Spanish. Output only the translation, nothing else.

   Text:
   ${output}
   ```
   (`${output}` is the required placeholder token.)
8. Set the **Selected Prompt** dropdown to the new prompt (creating ≠ selecting).

### Usage: two different hotkeys

- **Normal transcription hotkey** → plain transcript, no translation.
- **Post-Processing Hotkey = Ctrl + Shift + Space** → routes transcript through Ollama (translates).
- Getting English out instead of Spanish almost always means the normal key was used instead of Ctrl+Shift+Space.
- First call after idle is slow (model loads into VRAM); subsequent calls are fast.

---

## Next steps

- [x] Confirm `bun tauri dev` reaches a successful build + opens a window — **DONE**
- [x] Confirm plain transcription into a text field (audio path good) — **DONE** (Parakeet)
- [x] Wire Custom post-process provider to local Ollama, translate prompt, test end to end — **DONE** (qwen3:8b, EN→ES verified)
- [ ] Decide: utterance-based (Handy already does this) vs simultaneous/streaming (needs a different architecture — Handy is VAD-gated push-to-talk, no streaming partial-decode loop)
- [ ] Autostart Ollama/WSL so translation works without manually ensuring the server is up
- [ ] Try non-European input → switch transcription to Whisper (CPU-only after the Vulkan drop) and test
- [ ] Later, for standalone deploy: `bun tauri build` → installer; pre-stage the model file so the target machine is fully offline
