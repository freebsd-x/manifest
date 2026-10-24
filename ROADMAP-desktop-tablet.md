# FreeBSD-X — Desktop & Tablet Platform Roadmap

**Target:** desktop-primary (peers: macOS, Windows, ChromeOS), with a tablet
track (peer: iPadOS). Interactive version:
<https://claude.ai/code/artifact/b97cb7f7-e4b5-483b-801d-8f56de0f09a0>

The feature surface is strong — Vulkan/DRM compositor, HDR/VRR/HiDPI, a real
text stack, ~30 apps. The distance left to being a *platform* is architecture
maturity, security depth, QA, and a tablet layer that doesn't exist yet — not
more features. Roughly 80% of the way to a demoable desktop, ~15% to a platform.

## Where we stand

| Pillar | Status |
|---|---|
| Compositor & display (Vulkan/DRM, HDR, VRR, HiDPI, DMA-buf) | Shipped |
| Text & typography (HarfBuzz/FreeType, variable fonts, RTL, IME) | Shipped |
| Service bus (XBus, SEQPACKET + MSG_EOR framing) | Shipped |
| App suite (~30 apps) | Shipped (breadth) |
| Touch / stylus / gesture input | Partial (events exist; no touch-first UX) |
| On-screen keyboard (virtual_keyboard_service) | Partial |
| Sandboxing (Capsicum) | Partial (no runtime permission consent) |
| Session / auth / seat | Partial (prototype setuid helper, single seat) |
| Adaptive / responsive layout in xkit | **Gap** (foundational tablet blocker) |
| Display auto-rotation | **Gap** (RotationLock is a bare toggle) |
| Tablet-mode UX switch | **Gap** |
| QA & test coverage | Gap (improving) |
| SDK / ABI (versioned lib, installable) | In progress (see Phase 0 status) |
| Updates / OTA / telemetry | Gap |

## Phases (dependency order)

### Phase 0 — Productionize the core (prerequisite, both)
- **A. Versioned shared-library SDK + stable ABI.** Versioned `.so`, frozen wire
  protocol, installable headers + pkg-config so out-of-tree apps build against it.
- **B. QA as a pillar.** All test suites in CI, sanitizers, coverage floor,
  fuzzing the IPC/protocol parsers.
- **C. Crash + telemetry aggregation.** Real signal/backtrace capture, dedup.

### Phase 1 — Desktop-grade (desktop)
- Session/auth/seat hardening (real login, lock, multi-seat).
- Runtime permission system (camera/mic/location/screen-capture consent) on
  top of the Capsicum sandbox.
- Window-management depth (tiling layouts, multi-monitor hot-plug, arrangement
  UI, per-display scale).
- Accessibility that drives a real screen reader; i18n framework; laptop
  power/thermal robustness.

### Phase 2 — Tablet-grade (tablet)
- **Adaptive layout engine in xkit** — breakpoints, orientation-aware reflow,
  touch-target sizing. The gate for the whole tablet track; also improves
  desktop multi-monitor/fractional scaling, so start it early.
- **Auto-rotation pipeline** — accelerometer → DRM_MODE_ROTATE → input transform
  → reflow.
- On-screen keyboard maturation; touch-first shell & gestures; tablet-mode
  switch; split-view / slide-over; stylus depth.

### Phase 3 — Platform & ecosystem (both)
- SDK docs + ABI stability guarantee; signing/trust chain + distribution;
  OTA / A-B updates + rollback; (optional) cloud identity/sync/backup.

## The one sequencing bet

Phase 0 is non-negotiable and comes first — every durable thing assumes a
versioned SDK and a QA net. Then desktop-grade leads, but pull **one** tablet
item forward: the adaptive layout engine. It gates the tablet track and
simultaneously fixes desktop multi-monitor/fractional scaling, so it earns its
keep on the primary target while de-risking the secondary one.

## Phase 0 execution status (2026-09-16)

Done and committed across the repo workspace:

- **A1 — versioned library** (`build`): `libfreebsdx-ui` now installs as
  `.so.1 → .so.1.0.0` with SONAME; apps record versioned NEEDED. Policy in
  `docs/ABI.md`.
- **A3 — wire-protocol freeze** (`include`): compile-time `static_assert`s pin
  `XBusMessage`/`InputMessage`/`RenderMessage` size + offsets in every TU.
- **A2 — installable SDK** (`build`): `install_headers` + `pkgconfig.generate`
  (`freebsdx-ui.pc`). Verified by building a real app against the staged install.
- **B1/B2 — CI** (`manifest`): CI runs unit + smoke suites; added an ASan/UBSan
  sanitizer job. *Caveat:* CI runs from the manifest repo, so it needs a
  `repo init && repo sync` bootstrap step to assemble the workspace before
  building — follow-up.
- **B4 — fuzzing** (`sdk/xkit`): libFuzzer harness on `XBusClient::HandleIncoming`
  (verified 4.77M inputs, zero crashes).

Not yet done: `-Werror` enablement (needs a ~33-item warning cleanup, 6 of which
are in `apps/*` repos), A2 tail (convert a real `.xapp` to build standalone
against the SDK), B3 coverage floor, C1/C2 crash handler.
