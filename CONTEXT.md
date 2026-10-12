# PORPOISE: handoff context (paste this whole file at the start of every new AI chat)

> Porpoise = a smaller cousin of Dolphin that lives in a browser tab.
> Goal: run GameCube/Wii games fully client-side (static hosting, no server).
> Do NOT put the trademarked console name in the product name, repo name or UI.

## 0. First message to send in a new chat
"You are continuing project Porpoise. Read the attached CONTEXT.md fully (ROADMAP.md too, only the current phase).
Do ONLY the item under NEXT ACTION (section 2). Follow Session Rules (section 5).
Update CONTEXT.md after every finished step, not just at the end."

(Detailed phase specs live in ROADMAP.md. Do NOT read all of it. Only read the
section for the current phase: `grep -n "^## Phase" ROADMAP.md`, then Read with offset/limit.)

## 1. Mission and honest success tiers
- T1: homebrew .dol boots, frames on canvas (interpreter + software renderer). Proves the pipeline.
- T2: GameCube/Wii retail games run playable-but-slow on a strong desktop (WebGPU renderer, audio, input).
- T3: full speed on good hardware via a PowerPC->WASM JIT. The hard "think big" part.
Never claim a tier is reached without a measurement in section 3.

## 2. STATUS (keep current; this is the most important section)
- Phase: 0 (foundation)
- Last session: none yet
- NEXT ACTION: Phase 0 step 1. Create repo skeleton (section 6), add `dolphin` as a shallow git submodule,
  add GitHub Actions workflow that builds in the `emscripten/emsdk` docker image. Do not try to fix compile errors yet.
- Blockers: none known
- Needs the human: (list things only the human can do, e.g. "push to GitHub", "test in Chrome and paste status.json")

## 3. Measurements (update when they change; keep to one line each)
- Boots homebrew: no | Frames rendered: no | Speed (VI/s, 60 = full): n/a | Renderer: none | JIT: none

## 4. Architecture decisions (don't relitigate without a measured reason)
D1. FORK DOLPHIN, don't rewrite. A from-scratch Wii emulator (Broadway, Hollywood/GX, DSP, IOS) is years of work.
    Dolphin is GPL-2.0-or-later, so Porpoise is GPL too. Keep upstream merges possible: guard all changes with
    `#ifdef __EMSCRIPTEN__` or a `PORPOISE_WEB` define, keep patches small and in separate commits.
D2. PRIOR ART (verified Oct 2026, re-verify before depending on it): github.com/bitaxislabs/DolphinWeb and
    github.com/bitaxislabs/dolphin = Emscripten fork, interpreter + software renderer to WebGL2 canvas, DolphinNoGUI
    frontend, GC controller via touch/Bluetooth. Also a public WIP demo of a Dolphin WASM + WebGPU backend (Chad Hyatt).
    Use as a reference/starting diff, but own our fork so we can change anything.
D3. BUILD IN CI, not in chat sandboxes. Chat sandboxes usually only reach npm/pip/GitHub, so emsdk downloads may fail.
    GitHub Actions on `emscripten/emsdk` image builds the WASM and uploads artifacts + a short log tail.
    Test in Actions with Playwright + headless Chromium (software WebGPU/WebGL).
D4. LEGAL-SAFE TESTING: tests use only homebrew built in CI with devkitPPC (docker `devkitpro/devkitppc`) and Dolphin's
    hwtests. Never ask for, store or commit commercial games, keys or firmware. Users load their own files locally; files never leave the browser.
D5. CPU ladder: interpreter -> cached interpreter -> tiered PPC->WASM block JIT (hot blocks only).
    JIT emits WASM bytes in C++, instantiates in a worker, links via a shared funcref table + call_indirect. Paired singles use WASM SIMD128.
D6. GPU ladder: software renderer (baseline) -> WebGPU backend modelled on the Vulkan backend (WGSL from Dolphin's uber shaders).
D7. Threads: Emscripten pthreads (SharedArrayBuffer). Needs COOP/COEP headers; use a service-worker shim for hosts that can't set headers.
    Start single-core CPU/GPU, go dual-core later.
D8. ROMs: Emscripten WORKERFS mounts a user-picked File with zero copy and sync reads in workers. Saves/NAND go to OPFS (or IDBFS fallback).
D9. FEEDBACK LOOP: the page has a debug overlay with a "Copy status" button producing a tiny JSON
    (build id, fps/VI rate, PC, last log lines, GPU backend). The human pastes that into chat. Keep it under 40 lines.

## 5. SESSION RULES (context budget: the human is on the FREE plan, so be frugal)
1. One session = ONE task from NEXT ACTION. Stop after it. Don't start phase work "while you're here".
2. Update section 2 and section 7 immediately after each completed step (a session can be cut off at any time).
3. Never `cat` large files or logs. Use `tail -n 40`, `grep -m 20`, Read with offset/limit. Redirect build output to a file and show only the failing lines.
4. Never read the Dolphin tree broadly. Grep for symbols. Clone with `--depth 1` (and `--filter=blob:none` if possible).
5. No recaps, no re-explaining the plan, no apologies. Final reply is at most ~6 lines: what changed, what is next, what the human must do.
6. Batch edits; don't re-read a file you just wrote. Don't paste whole files into chat; write them to disk.
7. Delegate wide searches to a subagent if available and keep only its conclusion.
8. Prefer small testable steps: build -> run in CI -> look at a 40-line result -> fix. Do not guess at big refactors blind.
9. If a step fails twice the same way, STOP: record it in section 8 with what you tried, and write the NEXT ACTION as a diagnosis task.
10. Wrap up when you estimate roughly half the session budget is used. Leave NEXT ACTION precise enough for a cold start.
11. Keep this file under ~250 lines. Move stale log entries into docs/LEARNINGS.md.

## 6. Planned repo layout
```
porpoise/
  CONTEXT.md  ROADMAP.md           # this handoff system
  dolphin/                         # submodule: our fork, minimal #ifdef-guarded diffs
  web/                             # shell: index.html, main.ts, input, audio worklet, file loading, debug overlay
  jit/                             # PPC->WASM compiler (Phase 4), unit-testable standalone
  tests/homebrew/                  # devkitPPC test programs (built in CI)
  tests/e2e/                       # Playwright runner + frame-hash checks
  tools/                           # build scripts, status.json schema
  docs/LEARNINGS.md                # pitfalls, benchmarks, dead ends
  .github/workflows/               # build + test + deploy (GitHub Pages)
```

## 7. Phase checklist (details in ROADMAP.md)
- [ ] P0 Foundation: repo, submodule, CI skeleton, status overlay stub
- [ ] P1 Baseline build: reproduce an interpreter + software-render WASM build, boot a homebrew .dol (T1)
- [ ] P2 Shell: file loading, input, audio worklet, saves, COOP/COEP, deploy
- [ ] P3 WebGPU renderer
- [ ] P4 PPC->WASM JIT (T3 work)
- [ ] P5 DSP/audio quality
- [ ] P6 Wii specifics: IOS/HLE, disc formats, Wiimote, WebHID
- [ ] P7 Compatibility, mobile, polish, shader precompile

## 8. SESSION LOG (append-only, max 3 lines per session; newest last)
- (none yet)

## 9. LEARNINGS / PITFALLS (append; one line each)
- Chrome limits synchronous WASM compile on the main thread, so JIT compile/instantiate must happen in a worker (verify when reaching P4).
- WebGPU dual-source blending is a feature that may be missing; Dolphin has fallback blend paths. Check `adapter.features` first.
