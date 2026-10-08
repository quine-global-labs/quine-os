# Quine OS: build & CI guidelines

These guidelines describe how build tooling and CI should be structured
across Quine OS's apps and subsystems. They're drawn from, and should stay
consistent with, patterns already established in swap-watchdog, quine-linux,
quintodrome, and hyper.

## 1. Every app gets a Taskfile

Every standalone app (buildable, runnable, testable on its own) ships a
`Taskfile.yml` as its entrypoint — `task build`, `task install`, `task test`,
etc. Taskfile is the human-friendly front door; it is **not** where build
logic lives (see §3). Dev-only apps with no real build step just wrap the run
command directly (e.g. a Flutter app's Taskfile can be just
`flutter run -d linux` — nothing more needed).

## 2. Build tooling preference order

When an app needs a real build step, prefer in this order:

1. **Nix, as a hermetic dependency-provisioning layer inside the Dagger
   container** — not necessarily `nix build`/flakes, but `nix-shell -p
   <pkgs>` to pull build-time headers/toolchains into an ephemeral
   container, while the actual compile still runs the language's native
   builder (`go build`, `cargo build`, etc.). For example: Nix supplies
   `gtk3-devel`/`dbus-devel`-equivalent packages inside the container, the
   host's own Go toolchain is mounted in to avoid version drift against
   go.mod, and `patchelf` retargets the ELF interpreter so the binary still
   links against the host's real runtime libs. Nothing gets installed on
   the host or layered into the image for a one-off build dependency.
2. **If Nix genuinely can't cover it, the next-most-hermetic sandbox** —
   e.g. a `flatpak-builder` sandbox in place of Nix, when a dependency's
   header/binding generation isn't well supported in nixpkgs. That's a
   legitimate, documented exception, not drift.
3. **Never**: reaching for `rpm-ostree install`, `dnf`, or any host/image
   mutation just to satisfy a build-time dependency. If a dependency
   genuinely belongs in the OS image (not just at build time), that's a
   deliberate rpm-ostree layering decision made separately, not a build
   script shortcut.

## 3. CI: Dagger owns the logic, Gitea Actions stays thin

One Forgejo instance, Gitea Actions as the trigger, Dagger as the engine.
The workflow YAML should do as little as possible:

```yaml
- uses: actions/checkout@v6
- run: curl -fsSL https://dl.dagger.io/dagger/install.sh | BIN_DIR="$HOME/.local/bin" sh
- run: dagger call ci
```

All real logic — build steps, test invocation, lint — lives in `.dagger/`
(Go SDK), never inlined as shell in the workflow file. If a workflow step is
doing anything beyond checkout, installing the Dagger CLI, and one
`dagger call`, that's a signal the logic should move into the Dagger module
instead.

## 4. Apps vs. OS subsystems: different rigor, same tools

- **Apps**: full Taskfile, Dagger-driven CI as above, real test coverage
  expected — they're independently buildable and testable, so there's no
  excuse not to.
- **Larger OS subsystems** (not independently buildable/testable the way an
  app is — e.g. quine-linux itself): still require a Taskfile, and CI
  through *either* Dagger *or* a Taskfile-driven Gitea Actions job — not
  mandatory Dagger. Some kind of test/smoke-check should exist, but it's
  scoped to what's actually checkable (`bootc status`, image boots, a
  precondition script), not full unit/integration coverage.
- **Privileged/hardware-bound operations are a legitimate reason to skip
  Dagger entirely.** An ISO build needing root podman/buildah,
  `bootc-image-builder --privileged`, a specific runner label, and tens of
  GB of disk doesn't containerize cleanly inside Dagger's own container
  model. Driving that build directly via Taskfile + CLI tools, with the
  reasoning commented inline, is the right call — document the "why no
  Dagger here" rather than treating it as a gap to close later.

## 5. The cross-platform desktop exception

Dagger's container model is fundamentally Linux/OCI. It has no path to
macOS code signing/notarization, Windows `signtool`, or per-platform native
module rebuilds (e.g. node-pty, Tauri's Rust toolchain) — those need the
real OS. Cross-platform desktop apps (Quintodrome, Hyper) are the explicit
exception to "everything through Dagger":

- Keep Dagger for whatever portion *is* containerizable — a Go/Node
  server-side build, lint, or test job stays thin-Gitea-Actions-calls-Dagger
  as normal.
- The actual cross-platform packaging job gets a real OS matrix in Gitea
  Actions (`macos-latest`, `macos-13` for Intel, `windows-latest`) with
  native toolchain setup steps (Node, Go, Rust, mingw-w64 for Windows CGO) —
  this is allowed to be as "fat" as it needs to be, since there's no Dagger
  path to thin it against.
- Mark this divergence explicitly in the workflow (a comment explaining why
  Dagger isn't used there) so it reads as a deliberate, bounded exception
  rather than unexplained drift from the Dagger-first rule.
- Electron apps follow the same shape: native per-arch rebuilds and
  OS-specific signing/notarization mean packaging CI stays on real runners
  too, even once migrated off GitHub Actions onto Forgejo.

## 6. Mark Taskfiles and Dagger files as part of Quine OS

Since apps and subsystems get forked/vendored/copied around (and some
started life as upstream forks on their own hosting), every `Taskfile.yml`
and every `.dagger/main.go` should carry a one-line header identifying it as
a Quine OS file. This is purely an identifying tag — not a license or
copyright notice — so it stays short and makes no legal claim.

`Taskfile.yml`, as the first line:

```yaml
# Part of Quine OS. See BUILD_AND_CI.md in the quine-os repo for conventions.
version: "3"
```

`.dagger/main.go`, as the leading comment above `package main` (ahead of
any file-specific build-rationale comment already there):

```go
// Part of Quine OS. See BUILD_AND_CI.md in the quine-os repo for conventions.
package main
```

If a file already has a longer header comment explaining its own build
rationale (e.g. swap-watchdog's `.dagger/main.go`), add this as the first
line of that comment block rather than a second, separate block.
