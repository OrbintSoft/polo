# Polo — Modernization analysis and (hypothetical) development plan

> Analysis document. It does not describe work already done: it is a
> feasibility assessment and a proposed roadmap. Effort estimates are orders of
> magnitude, not commitments.
>
> Analysis date: 2026-06-06 · Code version analyzed: 19.0.0-alpha.1

---

## 1. Project snapshot

- **~75,000 lines** of Vala across **135 files**.
- Three-layer architecture:
  - **UI** — 67 files in `src/Gtk/`, GTK 3 + embedded VTE 2.91 terminal.
  - **Logic / utility** — 49 files in `src/Utility/`, including the shared
    "TeeJee" library by the original author (the same base as Timeshift).
  - **Console helper** — 7 files (`src/polo-disk.vala`) for disk management,
    already built as a separate binary.
- Monolithic files: `src/Gtk/FileViewList.vala` (**6,259 lines**),
  `src/Utility/Device.vala` (~3,000), `src/Gtk/Settings.vala` (~3,000),
  `src/Gtk/FileContextMenu.vala` (~2,900).
- **942** `connect` calls on GTK signals · **44** custom widget classes ·
  **~218** menubar actions · **~24** windows/dialogs/popovers.
- **Zero tests** · no test CI · no framework.

### Runtime dependencies
`glib-2.0`, `gio-unix-2.0`, `posix`, `gtk+-3.0`, `gee-0.8`, `json-glib-1.0`,
`libxml-2.0`, `vte-2.91`, `gdk-x11-3.0`.

### Nature of the code: an orchestrator of external tools
Most of the "rich" features are not the app's own logic but **calls to system
binaries** (57 `exec_script`):

| Feature             | Delegates to |
|---------------------|--------------|
| Archives            | `tar`, `7z`, `bzip2`, `xz`, `lzop`, `gzip` (200+ references) |
| Disks/mount/LUKS    | `udisksctl`, `cryptsetup`, `mount`, `gvfs` |
| Checksums           | `md5sum`, `sha1sum`, `sha256sum`, `sha512sum` |
| PDF/images/video    | `pdftk`, `convert`, `ffmpeg`, `exiftool`, `youtube-dl` |
| Antivirus           | `clamscan` |

This is a double-edged sword:
- **Pro** — the complex logic does not need to be rewritten: it is enough to
  reproduce how the commands are built and how their output is parsed.
- **Con** — everything is glued to GTK/GLib/GIO/Gee. That is where the real
  cost of any port lies.

---

## 2. Cross-cutting obstacles

### 2.1 The global `App` singleton
In `src/Common/Main.vala`: `public Main App;` — a global object used by
**59 of the 135 files**. It is the shared state that holds the application
together and is the main blocker for:
- **testability** — you cannot instantiate a module in isolation without
  dragging in half the app;
- **modularization** — modules are not independent;
- **incremental porting**.

Breaking this dependency is the enabling refactor for everything else.

### 2.2 Model/UI mixing
The data model `src/Utility/FileItem.vala` contains `Gdk.Pixbuf` and icon
caching: UI is baked into the model.

### 2.3 Monster files
`FileViewList.vala` (6,259 lines) mixes UI, logic and state in a single file.

### The good news
**32 of the 49 files in `src/Utility/` have no Gtk/Gdk reference** — they are
near-pure logic (archives, rsync, pdf, images, mount, parsing, checksum). That
surface is already "almost modularized" and immediately testable.

---

## 3. Converting to a more mainstream language

**Obstacle #1 is not the language: it is GTK.** Vala is essentially GTK/GLib
with C#-like syntax. The bulk of the effort is the dependency on the GObject
runtime, not the syntax.

| Language | Toolkit | Assessment |
|----------|---------|------------|
| **Rust** ⭐ | `gtk-rs` + `glib-rs` (mature GTK3/4 + VTE bindings) | Technically best: you stay on GTK, UI translates ~1:1. Friction: Rust's ownership model vs the cyclic GObject graph and 942 callbacks → lots of `Rc<RefCell>`/`clone!`. |
| **C++** | `gtkmm` | More "literal" port, least impedance mismatch. More legacy. |
| **C#** ⚠️ | `GtkSharp`/`gir.core` | Vala→C# is almost mechanical syntactically, **but** the GTK bindings are less maintained/complete, especially VTE (the F4 embedded terminal, a key feature). The riskiest. |

In all cases **you stay on GTK**. Rewriting the UI on a cross-platform toolkit
(see §7) multiplies the cost 3-4x.

### Effort estimate for a full port

| Block | LOC | Difficulty |
|-------|-----|-----------|
| TeeJee utilities (FS, Process, Json, Logging) | ~5k | Low |
| Console/disk helper | ~3k | Medium |
| Archive/file/device logic | ~12k | Medium |
| GTK UI layer (67 files) | ~45k | **High** (bulk of the risk) |

**Order of magnitude: 6-12 person-months** for a complete, stable port.

### Can Claude Opus do it?
Partially yes, fully in one shot no:
- **Good fit**: porting utility/console modules, mechanical file-by-file
  translation, project scaffolding, converting widget classes one at a time.
- **Problematic**: scale (75k lines » a single context session → dozens of
  tasks with checkpoints); long build-fix cycles on subtle GTK errors; lack of
  a "ground truth" without tests.
- **Right approach**: an initial PoC (TeeJee + 2-3 windows + VTE terminal) to
  validate the chosen binding, *then* decide.

---

## 4. Testing strategy

### 4.1 100% coverage: the wrong goal
Unrealistic and counterproductive on a GTK app. 60% is UI: covering it 100%
means testing GTK more than your own code, with fragile tests.

**Sane goal**: ~70-80% on the **logic** (the 32 pure files + the models), UI
left to smoke tests + selective E2E.

| Target | Effort |
|--------|--------|
| 70-80% on utility/console logic (~20k lines) | weeks |
| Coverage of stateful models (requires breaking `App` first) | +1-2 weeks |
| Toward 100% including UI | months, negative ROI (not recommended) |

### 4.2 GitHub CI — feasible (1-2 days)
- Build on an Ubuntu container with `valac` + the `--pkg` deps.
- Logic tests: no display needed.
- Tests touching GTK: `xvfb-run` (headless virtual X), the standard pattern.

Do this **first**: an immediate safety net on the build.

### 4.3 E2E tests — possible but fragile
- Via AT-SPI with `dogtail`/`ldtp` (drive widgets through the accessibility
  tree). Requires accessible names on widgets, **probably absent today**.
- More robust alternative: **integration tests on the logic** (e.g. create an
  archive in a temp dir → verify with `tar -t`). End-to-end behavior, no UI.
- Verdict: a few "smoke" E2E on critical flows + many logic integration tests.

---

## 5. Manual test document (QA plan)

### Size
- ~218 menubar actions · ~24 windows/components · context menu · physical
  interactions (drag&drop, multi-pane/tab, inline rename).
- Estimate: **150-250 manual test cases**. A sizeable but feasible document.

### Feasibility: yes
1. The source of truth **exists in the code** and is mechanically enumerable
   (menubar actions, windows, context-menu entries).
2. It can be generated systematically, with human review for usage details.

### Three tiers
| Tier | Example | Test |
|------|---------|------|
| **A** — logic, automatable | create archive, sha256, parse device | integration test (out of the manual doc) |
| **B** — manual, deterministic | open CreateArchiveWindow, set options, create, verify | step-by-step (core of the doc) |
| **C** — manual, perceptual/environmental | drag&drop, thumbnails, F4 terminal, physical mount, LUKS, cloud login | steps + human verification, depends on hardware/network (the irreducible part) |

The manual document = **B + C**. Tier **C** is what can *never* be reliably
automated and must be isolated and listed explicitly.

Schema per case: `ID · Area · Preconditions · Steps · Expected result ·
Automatable? (A/B/C) · Depends on hardware/network?`

### Risks
- **Staleness** — mitigate by anchoring each case to the file/action in the code.
- **Environmental preconditions** — USB device, LUKS, cloud account, SMB/FTP server.
- **Granularity** — 218 actions ≠ 218 tests: group the variants.

---

## 6. Hybrid Rust + Vala port — feasible ✅

Technically real and lower-risk than a big-bang rewrite.

### Mechanism (standard GObject infrastructure)
- Vala compiles to C and exposes everything as **GObject** with `.vapi`/`.gir`.
- Rust, via **gtk-rs / the `gir` system**, consumes GObject libraries (this is
  how gtk-rs itself talks to GTK).
- A Vala module → shared GObject library → called from Rust like any GObject
  library. **And vice versa**: Rust modules exposing GObject used by Vala.

### Strategy
1. Modularize (see §2).
2. Take a **leaf, pure-logic, GTK-free** module (e.g. checksum or archives) and
   rewrite it in Rust as a C/GObject library.
3. The rest of Polo (Vala) calls it without noticing.
4. Migrate one module at a time, **app always buildable and working**.

### Honest friction
- The boundary goes through the C ABI / GObject: GObject types pass fine, Gee/
  idiomatic-Rust types must be wrapped → migrate whole modules along clean
  boundaries.
- More complex build system (`valac` + `cargo` + `.gir` + linking) → Meson
  handles both, a few days of initial setup.
- The GTK UI is best **left last in Vala** (it is the verbose part in Rust).

### Benefits
- Gradual migration without stopping the app.
- Each Rust module testable in isolation (excellent native test support).
- **You can stop whenever you want**: even a permanent hybrid (Rust logic +
  Vala/GTK UI) is a valid end state.

---

## 7. The "drop GTK + cross-platform" goal

This is the **most expensive** option: a from-scratch UI rewrite on a
cross-platform toolkit. It only makes sense as a final phase, **after** the
logic is in Rust and tested — at that point it would "just" be attaching a new
UI to a clean core.

---

## 8. Hypothetical development plan (by phases, increasing risk)

Each phase is a valid stopping point: you can stop at any phase with a working,
improved product.

### Phase 0 — Safety net (days)
- [ ] GitHub Actions CI: Vala build + `xvfb-run` for tests that touch GTK.
- [ ] Smoke test: the app starts, browses a folder, opens the terminal.
- **Outcome**: every commit verifies the build does not break. Zero code risk.

### Phase 1 — Inventory and QA plan (days)
- [ ] Extract into a raw table: 218 actions + 24 windows + context menu.
- [ ] Classify each entry A/B/C (see §5).
- [ ] Draft the manual test document (tiers B+C), anchored to files.
- **Outcome**: reference spec for any future port + an index of the work.

### Phase 2 — Tests on pure logic (weeks)
- [ ] Test suite over the 32 GTK-free files in `src/Utility/`.
- [ ] Golden tests on command construction (`tar`, `7z`, `udisksctl`...).
- [ ] Integration tests on real flows (create archive → verify contents).
- **Target**: ~70-80% coverage on the logic.

### Phase 3 — Modularization (weeks/months) — enabling refactor
- [ ] Extract a **core** module (filesystem, process, json, logging) free of `App`.
- [ ] Convert the **tasks** (archive, rsync, pdf, image, mount) to dependency
      injection instead of the global `App`.
- [ ] Separate model from rendering (remove `Gdk.Pixbuf` from `FileItem`).
- [ ] Split the monster files (`FileViewList.vala`).
- **Outcome**: modules with clean boundaries, testable, ready to port.

### Phase 4 — Hybrid Rust port, module by module (months)
- [ ] PoC: rewrite a leaf module (checksum or archives) in Rust as a GObject
      library, integrate it into the Vala app. **Measure the real cost.**
- [ ] Progressively migrate the logic modules, app always working.
- [ ] Cover each Rust module with native tests.
- **Outcome**: tested Rust core, UI still in Vala/GTK. A valid end state.

### Phase 5 (optional, long term) — cross-platform UI
- [ ] Only after Phase 4: a new UI on a cross-platform toolkit over the Rust core.
- **Outcome**: cross-platform Polo. Maximum cost, to evaluate once the core is ready.

---

## 9. Summary of recommendations

| Priority | Action | Cost | Risk | Value |
|---------|--------|------|------|-------|
| 1 | CI + smoke tests (Phase 0) | days | none | high |
| 2 | Manual QA plan (Phase 1) | days | none | high |
| 3 | Pure-logic tests (Phase 2) | weeks | low | high |
| 4 | Modularization (Phase 3) | weeks/months | medium | decisive |
| 5 | Rust hybrid (Phase 4) | months | medium | high |
| 6 | Cross-platform UI (Phase 5) | months+ | high | strategic |

The ideal candidate to **start and measure the real costs**:
CI + a test suite for the **checksum** or **archive** module (small, isolated,
gives an immediate concrete number for the rest).