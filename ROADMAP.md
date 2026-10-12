# PORPOISE ROADMAP (reference only; read just the current phase)

Each phase: Goal / Steps / Acceptance test / Risks. A step should fit in one short AI session.

## Phase 0: Foundation
Goal: a repo where every later session can build, test and report without the human doing anything but pushing.
Steps:
1. Repo skeleton (CONTEXT.md section 6), LICENSE (GPL-2.0-or-later), README stating "bring your own legally dumped games".
2. `git submodule add --depth 1` the Dolphin fork (start from bitaxislabs/dolphin if reachable, else upstream dolphin-emu/dolphin).
3. CI workflow: job `build-wasm` in `emscripten/emsdk`; upload `.js/.wasm/.data` + `build.log.tail` (last 80 lines) as artifacts.
4. CI job `homebrew`: build a tiny devkitPPC .dol (colored screen / spinning triangle / text) for tests.
5. `web/` stub page + debug overlay + `status.json` schema (CONTEXT D9).
Acceptance: CI is green on an empty-ish build; artifacts download; overlay renders locally.
Risks: Dolphin has many third-party deps (CMake externals); expect to disable GUI/Qt, Vulkan, Android, netplay, discord, etc.

## Phase 1: Baseline build -> T1
Goal: Dolphin core compiles to WASM, runs the CPU interpreter, software renderer pushes frames to a canvas.
Steps:
1. Diff the prior-art fork against upstream (`git diff --stat`) to learn which areas needed patches. Summarize in docs/LEARNINGS.md (short).
2. Reproduce: CMake with Emscripten toolchain, DolphinNoGUI-style frontend, no JIT arch, `SingleCore`, software video backend.
3. Replace platform bits: windowing -> canvas; threads -> pthreads pool; timing -> `emscripten_get_now`; fastmem mmap -> plain arrays/software MMU.
4. Frame out: copy XFB/EFB result into a shared buffer; JS draws to canvas (WebGL2 texture upload or 2D `putImageData`).
5. Boot the CI homebrew .dol. Playwright asserts a frame hash is non-blank and stable.
Acceptance: Playwright run in CI shows the homebrew's expected frame. Record VI/s in CONTEXT section 3.
Risks: static initializers/ordering, 32-bit pointer assumptions (wasm32), big binary size (use `-Os`/LTO later), sync file IO in main thread.

## Phase 2: Shell
Goal: a usable web app around the core.
Steps: file picker + drag/drop + WORKERFS mount; Gamepad API + keyboard map; mouse = Wii pointer; AudioWorklet fed by a SAB ring buffer (stub tone first);
OPFS save storage; COOP/COEP via `_headers` + service-worker fallback; GitHub Pages deploy; settings panel; save states.
Acceptance: user picks a .dol/.iso locally, sees video, pad input reaches the core, nothing is uploaded anywhere.
Risks: iOS Safari quirks (HTTPS required for SAB), memory limits on mobile, large ISO (4.7GB+) handling via `File.slice`, never load whole into RAM.

## Phase 3: WebGPU renderer
Goal: replace the software renderer with a GPU backend (the Phase 2 -> T2 jump).
Steps:
1. Skeleton `VideoBackends/WebGPU` implementing the `AbstractGfx`/`VertexManager`/`TextureCache` interfaces, modeled on the Vulkan backend.
2. Device init, swapchain on the canvas (OffscreenCanvas in a worker if possible), command encoder lifecycle.
3. Shader path: translate Dolphin's generated uber-shader pipeline to WGSL (a GLSL->WGSL step via naga/tint in CI, or a direct WGSL generator). Prefer uber shaders to avoid stutter from per-config compiles.
4. EFB: color/depth targets, EFB pokes/peeks (can start as slow readbacks, game-dependent), XFB copy, scaled output.
5. Textures: format decoders (CMPR/I4/IA8/RGB565...) on CPU first, GPU decode (compute) later.
6. Blend modes, depth, fog, TEV stages; missing-feature fallbacks (dual-source blend).
Acceptance: devkitPPC GX test programs (TEV, textures, fog, blending) match reference frame hashes; then run a hwtests subset.
Risks: WebGPU is async; the core expects sync GPU readbacks. Design a non-blocking path, drop or approximate EFB peek at first.

## Phase 4: PPC -> WASM JIT (T3)
Goal: 5 to 20x CPU speedup over the interpreter.
Design:
- Guest registers live in linear memory (a struct at a fixed address). Compiled code reads/writes them with loads/stores. Keep hot values in wasm locals within a block.
- Compile units: basic blocks first, then traces. Tiered: interpret until a block runs N times, then compile.
- Emit raw WASM binary from C++ (small emitter; reference ideas: v86's x86->wasm JIT, QEMU TCG-wasm).
- Instantiate in a worker (`new WebAssembly.Module` + `Instance` sharing imported memory), publish into a shared funcref table; dispatcher uses `call_indirect`.
  Batch several blocks per module to amortize instantiate cost. Evict via table slot reuse.
- Memory: software MMU fast path with BAT/page-table lookup inlined; MEM1/MEM2 direct. Check for self-modifying code: invalidate on writes to compiled ranges (page bitmap).
- FP: paired singles -> f32x4/f64x2 SIMD128; rounding/FPSCR/NaN semantics need care (test vs interpreter).
- Exceptions/timing: end blocks at DEC/interrupt checkpoints; downcount accounting like Dolphin's JIT.
Steps: (1) wasm emitter + unit tests in node (no browser needed), (2) integer ALU + branch subset behind a flag with differential testing against the interpreter,
(3) loads/stores + MMU, (4) FP + paired singles, (5) block linking, (6) tiering + eviction, (7) benchmarks.
Acceptance: differential fuzz (random instruction sequences, compare full register/memory state vs interpreter) passes at huge counts; homebrew benchmark VI/s improves.
Risks: compile latency, code size, debugging generated wasm (write a disassembler dump flag and use `wasm-tools`/`wasm2wat` in CI).
This phase is perfectly suited to AI: it is self-contained and verifiable by differential testing.

## Phase 5: DSP and audio
HLE AX ucode first (fast), LLE DSP interpreter for other ucodes later; resample to the AudioContext rate; latency tuning; underrun metrics in the overlay.

## Phase 6: Wii specifics
IOS HLE coverage (FS/ES/USB/SD stubs), disc formats (ISO, GCZ, WBFS, RVZ via zstd compiled to wasm), WAD/NAND setup flow with user-supplied data only,
Wiimote emulation (pointer, motion via keyboard/mouse; phone sensors via DeviceMotion; real Wiimote via WebHID, experimental), Wii Remote speaker audio, save data in OPFS.

## Phase 7: Compatibility and polish
Compat table (game-agnostic notes only, no game files), per-title fixes only as data-driven config, mobile layout, PWA install, shader/pipeline precompilation, perf HUD, docs, release.

## Standing test plan
- Unit tests in node for jit/ and utilities. No browser needed.
- Playwright e2e (headless Chromium) on every push: boot homebrew, frame hash, VI/s floor, no console errors.
- Differential CPU fuzzing (Phase 4+). Dolphin hwtests where applicable.
- Human tests commercial games locally and pastes the status JSON; AI never handles game files.
