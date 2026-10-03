# Offline Linux + TUI Support: Current State and Approach

This is a feasibility analysis, not an implementation. It maps what already exists
toward running Orca (a) on a Linux host with no internet access and (b) driven
primarily through a terminal UI instead of the Electron desktop app, and lays out
the smallest path to get there. No product code changes are included.

## Two separable problems

"Offline" and "TUI" are independent axes and should not be conflated:

- **Offline** is about which network calls the *runtime* makes on its own
  (telemetry, auto-update, skill installs) versus network calls the *AI coding
  agents it orchestrates* make (Claude, Codex, etc., which talk to their own
  cloud APIs unless pointed at a self-hosted/local model). Orca's runtime can be
  made to need zero network access; the agents it launches are a separate,
  per-agent decision the operator makes, outside Orca's control.
- **TUI** is about which client renders the UI: today every client
  (Electron desktop, the web/mobile companion, the CLI) is either a React-DOM
  renderer or a scripting interface — none is an interactive terminal UI.

## Current state

### The backend is already Electron-free — but not shipped

`src/main/orcad/` is "the Orca runtime served from plain Node, no Electron"
(`src/main/orcad/orcad-entry.ts:1-13`). It constructs the same
`OrcaRuntimeService` the desktop app uses, installs a headless PTY controller,
and serves the same runtime RPC — with no `BrowserWindow`, no native
notifications, no Xvfb, no GPU/Chromium libraries at all. This is enforced, not
aspirational: `config/scripts/check-runtime-electron-ratchet.mjs` bundles
`orca-runtime.ts`, `runtime-rpc.ts`, and `orcad/main.ts` with esbuild and fails
the build if any module reachable from the runtime imports `electron`.
`config/runtime-electron-baseline.txt` — the allowed list — is currently
**empty**, meaning the runtime graph as shipped today has zero Electron imports.

`orcad` is built with `pnpm run build:orcad` and exercised in CI by
`smoke:orcad-terminal`, but it is **not packaged or released**: no workflow
produces a distributable `orcad` artifact, and neither `README.md` nor any
`docs/readme/*` file mentions it. It is presently a dev/CI-internal build that
proves the architecture, not a product.

This matters because the currently *documented* headless path
([`headless-linux-server.md`](./headless-linux-server.md)) runs the Electron
**AppImage** in server mode (`orca serve`) — it still needs Xvfb (or a real
`DISPLAY`), the full Chromium shared-library set, and `libfuse2` for AppImage
extraction, even though nothing is ever shown on screen. `orcad` needs none of
that. Productizing `orcad` is the biggest single lever for making offline Linux
deployment lighter: no virtual display, no GPU library set, no AppImage/FUSE
step — just a Node process and native PTY/SQLite addons already gated behind
the glibc 2.31 floor ([`linux-glibc-compatibility.md`](./linux-glibc-compatibility.md)).

`orcad` currently ships "Variant B": the browser-pane and speech feature
clusters are excluded because they are the only modules that statically import
`node:sqlite`, which would otherwise raise the minimum Node version from 18 to
22.5+ (`config/scripts/build-orcad.mjs:1-9`). A TUI-first, offline deployment is
exactly the profile that can accept losing in-app embedded-browser preview and
speech features.

### The runtime protocol is already client-agnostic

Beyond Electron, Orca already has three independent clients talking to one RPC
surface:

- The Electron renderer (React DOM, `src/renderer/src/`).
- A **browser-based web client** (`vite.web.config.ts` builds
  `src/renderer/web-index.html` to `out/web`; client logic in
  `src/renderer/src/web/web-runtime-client.ts` and related files), used for
  mobile/web pairing. It talks to a runtime over WebSocket with no Electron
  dependency at all.
- The `orca`/`orca-ide` **CLI** (`src/cli/`), which is itself a scripting RPC
  client (`src/cli/runtime-client.ts`) that can start (`orca serve`), pair,
  list terminals, and drive orchestration — see the CLI examples throughout
  [`orcad-operations.md`](./orcad-operations.md) and
  [`headless-linux-server.md`](./headless-linux-server.md).

The RPC surface itself is large and generated:
`src/shared/rpc-contract/rpc-params-catalog.generated.ts` currently enumerates
more than 600 typed methods (`config/scripts/generate-rpc-params-catalog.mjs`).
Building a client against this protocol is proven, low-risk work — three
clients already do it — but matching full desktop feature parity in a fourth
(TUI) client is not realistic for a first version.

The terminal daemon (`orcad` forks it detached; see
[`orcad-operations.md`](./orcad-operations.md)) owns every local PTY over its
own Unix-socket protocol, independent of both Electron and the RPC/WebSocket
layer. This is the piece a TUI needs for byte-accurate interactive terminal
panes — attach to the daemon's socket directly rather than re-deriving
terminal semantics inside a TUI framework.

### Offline-readiness of what already exists

- **Telemetry** already has three independent, already-implemented kill
  switches, in this precedence: `DO_NOT_TRACK=1` (community standard),
  `ORCA_TELEMETRY_DISABLED=1` (product-specific), and CI auto-detection via
  `CI`/`GITHUB_ACTIONS`/etc. (`src/main/telemetry/consent.ts:26-109`). No code
  change is needed to run fully opted-out; an offline-deployment guide only
  needs to tell operators to set these.
- **Auto-update never runs in headless/serve mode.** Per
  [`orcad-operations.md`](./orcad-operations.md) and
  [`headless-linux-server.md`](./headless-linux-server.md#upgrade), "In
  headless mode Orca wires up no auto-updater at all — the built-in updater
  only runs in the desktop GUI." Nothing to disable for an offline deployment.
- **Push notifications to paired mobile clients silently no-op headless**
  (documented in `headless-linux-server.md`'s pairing-troubleshooting section)
  since agent-completion detection runs in the (unstarted) desktop renderer —
  expected, not a bug, for a server/TUI deployment.
- **Skill installs are the one clearly network-dependent operator flow left**:
  `orca skills install` shells out to `npx skills add <repo> --skill <name>
  ...` (see "Installing Agent Skills Without A Desktop" in
  [`headless-linux-server.md`](./headless-linux-server.md)), which needs npm
  registry reachability. There is no vendored/offline-mirror path today.
- **AI coding agents themselves are out of scope for Orca's own
  offline-ness.** Claude, Codex, and similar agents call their own cloud APIs
  unless configured against a self-hosted or local model; that is an
  operator/agent-configuration decision Orca does not currently mediate.

## What is missing

1. **`orcad` is not a distributable artifact.** No release workflow builds or
   publishes it; no docs tell an operator how to get it onto a Linux box.
   Packaging is the prerequisite for "runs on offline Linux without the
   Electron/Xvfb/FUSE stack."
2. **No TUI client exists.** Nothing in the repo renders a terminal UI; `ink`,
   `blessed`, and similar terminal-UI libraries are not dependencies. The
   nearest precedent is the CLI's scripted/JSON output, which is not
   interactive.
3. **No offline-hardening checklist or default posture.** The kill switches
   exist but are opt-in via env var; there is no single documented "air-gapped
   install" flow that sets them by default and calls out the skills-install
   gap.
4. **No offline path for `orca skills install`.** It hard-requires `npx`
   resolving packages from the npm registry.

## Recommended approach

Phased, smallest-first, each phase independently shippable:

1. **Package `orcad` for Linux.** Produce a release artifact (tarball or
   minimal container image) with no Electron/Chromium/Xvfb/FUSE dependency,
   built from the existing `pnpm run build:orcad` /
   `build:orcad-prebuilds` pipeline. Extend or supersede
   [`headless-linux-server.md`](./headless-linux-server.md) with an
   `orcad`-native deployment path once it exists, validated against the
   glibc 2.31 floor. This alone makes today's "offline Linux" story
   substantially lighter, with zero new UI work.
2. **MVP TUI as a new client package**, not inside `src/renderer` (keeps
   `check:runtime-electron-ratchet` meaningful and follows the existing
   host-port pattern in `src/main/host/`). Reuse, don't reimplement, per
   AGENTS.md's "Reuse Before Reimplementing": the same typed RPC client
   pattern `src/cli/runtime-client.ts` and the web client already use, and the
   generated `rpc-params-catalog`. Scope the first version narrowly: worktree
   and agent-session list, orchestration status, and one interactive terminal
   pane attached directly to the terminal daemon's existing socket protocol.
   `ink` (React reconciler targeting the terminal) is the natural rendering
   choice given the codebase's existing React/Zustand conventions, but the
   daemon's socket protocol is currently internal/undocumented for third-party
   consumption and needs an explicit, reviewed contract before a new client
   type depends on it directly — the same wire-compatibility discipline
   [`remote-wire-compatibility.md`](./remote-wire-compatibility.md) requires
   for any new stream consumer.
3. **Expand TUI coverage by RPC-catalog priority** (git status, orchestration
   launch/stop, session search) rather than chasing the full 600-plus-method
   desktop surface. Full parity with the Electron app is an explicit
   non-goal, matching how the existing web/mobile clients already diverge
   from desktop parity.
4. **Offline-hardening pass**: document (and consider defaulting, in an
   explicitly air-gapped install mode) the telemetry kill switches, and
   investigate a vendored or pre-fetched skills-install path so
   `orca skills install` has an offline story.

## Open risks

- The terminal daemon's socket protocol has no external contract today; a TUI
  that depends on it directly is a new, currently-undocumented wire consumer.
- `orcad`'s Variant B trade-off (no browser-pane/speech clusters, to hold the
  Node floor at 18) is likely acceptable for a TUI/offline-server audience but
  is a real feature gap versus the desktop app and should be stated up front,
  not discovered later.
- No prior TUI code exists in this repo to extend; this is new surface area,
  so it should start as the smallest possible vertical slice (one terminal
  pane + a session list) before any broader UI investment.
