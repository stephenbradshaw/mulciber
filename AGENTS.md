# AGENTS.md - build brief for Mulciber

This document is the working brief for AI sessions building and
maintaining this repository. It contains the full context a fresh session
needs: purpose, integration contract, toolchain facts, architecture plan,
and conventions. Read it fully before writing code. Host-specific and
deployment-specific details live in `LOCAL_NOTES.md` (gitignored - never
commit it).

## What this project is

A small HTTP service that compiles Beacon Object File (BOF) source code
into relocatable object files, so operators can submit BOF **source**
through an agent's file-upload flow and receive a compiled object,
instead of compiling by hand on their own machine.

The consuming agent runs BOFs with two API dialects (see "Integration
contract" below). Today the operator compiles objects manually (clang /
zig cc on their workstation) and uploads the finished object; this service
moves that step server-side and makes the obfuscating toolchain available
to every operator.

The service is named after Mulciber: Vulcan's smith epithet ("he who
melts metal") and, in Paradise Lost, the architect of Pandemonium.

## Repository posture

- **Public repository.** Keep everything in the repo suitable for public
  consumption: no secrets, no infrastructure hostnames, no credentials,
  no consumer project names beyond what is technically required to
  describe object formats and dialects. Deployment-specific values
  (hosts, tokens) go in environment variables or untracked files.
- **BSD 3-Clause license** (see LICENSE).
- **Commit messages: short one-liners.** Do not write long multi-paragraph
  commit bodies; the detailed rationale belongs in README/code comments
  and the changelog.
- Prefer small, focused commits.
- Before every push, check that no consumer project names, hostnames or
  local paths leaked into staged content.

## Integration contract

The consumer is a Rust C2 agent with a `coff` command that loads and
executes BOFs. Its loader implementation lives in the consumer repository
(see `LOCAL_NOTES.md` for the local checkout path); read `mod.rs`,
`beacon.rs`, `link.rs`, `coffobj.rs`, `elfobj.rs`, `win.rs`, `unix.rs`
there if you need to match loader-level details exactly.

### Dialects and formats

Two BOF API dialects are implemented by the agent loader:

1. **Cobalt Strike dialect** (Windows COFF BOFs): `go` entrypoint,
   `BeaconPrintf`/`BeaconOutput` with the classic 4-arg stack layout.
   Object format: PE/COFF relocatable (`clang --target x86_64-pc-win32
   -c`-style output).
2. **Dark-Agent dialect** (macOS/Linux ELF BOFs): `coffee(argc, argv)`
   entrypoint, `BeaconPrintf(char* fmt, ...)` with 8 variadic register
   slots and `BeaconOutput(const char* data, int len)`. Object format:
   **ELF64/i386/ARM relocatable**.

**Critical macOS detail:** the macOS loader executes **ELF** objects, not
Mach-O. macOS BOFs must be compiled as ELF relocatables, roughly:

```
zig cc -target aarch64-linux-none -I <darwin-headers>/usr/include \
    -D__aarch64__=1 -D__arm64__=1 -fPIC -c bof.c -o bof.o
```

(the exact incantation lives in the coff loader's error message in
`mod.rs`). The Darwin SDK headers are not available in the build
environment - see "Darwin headers" below.

### Architectures

- Windows COFF: x86 (`i386-pc-win32` style), x64 (`x86_64-pc-win32`),
  arm64 if practical.
- Linux ELF: x86, x64, armv7, arm64.
- macOS ELF: x64, arm64.

### Service surface

Expose (exact route naming is yours to finalize; keep a stable versioned
prefix, e.g. `/api/v1/...`, and note the outer reverse-proxy will mount
the service under a sub-path such as `/bofcompiler` - design for that
from day one, do not assume you are served at the root):

- `POST /api/v1/compile` - compile submitted source. Accepts: source
  file(s) (single file or zip/tar archive), target dialect, target
  architecture, obfuscation level (off/light/full), optional include
  files. Returns the compiled object file (binary download) plus build
  log. Failures return the compiler stderr so the operator can fix their
  BOF.
- `GET /api/v1/health` - liveness.
- Consider a `GET /api/v1/examples` or bundled library endpoint later
  (see "BOF library" below).

Authentication: a static bearer token from the environment (no user
database). Operators pass it through the C2 UI flow; the integration
design (below) is for the consuming agent's operator interface, not this
service's problem - but the service must require the token on every
route.

### Obfuscation levels

The obfuscating clang from the toolchain release carries the obfuscation
C passes. Map levels to pass sets (keep consistent with the consumer's
Rust obfuscation tiers where sensible):

- `off` - no passes
- `light` - `-sobf` (string obfuscation), `-mllvm -sub` (instruction
  substitution), `-mllvm -const` (constant encoding) - verify exact
  flag spellings against the toolchain release notes; the passes were
  added upstream as C-level obfuscation passes
- `full` - the above plus `-mllvm -split` (basic block splitting) and
  `-mllvm -bcf` (bogus control flow)

Caveat learned from the consumer's build system: BOF code is tiny, so
obfuscation output growth is small, but `-bcf`/`-fla` can bloat trivial
functions disproportionately - sanity-check output sizes per level and
document typical size multipliers.

## Toolchain facts

The service container should build FROM the shared compiler base image
rather than installing its own toolchain:

```
FROM ghcr.io/stephenbradshaw/obf-build-base@sha256:<pin>
```

The current digest pin lives in the consumer repository's Dockerfile
(see `LOCAL_NOTES.md`); keep the two in sync deliberately (a shared
constant both repos update together is fine).

The base image provides:

- The **obfuscating clang** at `$OBF_CLANG`
  (`/opt/llvm-obf-clang/current/bin/clang`), built from the forked LLVM
  23.1.3 with all codegen backends and the obfuscation passes. Release
  source: `https://github.com/stephenbradshaw/llvm-project/releases`
  (tag `v23.1.3.10` at the time of writing).
- **zig** (via cargo-zigbuild's bundled ziglang) - needed for the
  `-none`-targeted ELF compiles (macOS BOFs) and potentially as the
  linker for linux BOFs.
- Stock rustc/cargo - for the service implementation itself if Rust is
  chosen; the plan (below) says Python/FastAPI, so mostly irrelevant
  except that the image has it.

The service runtime itself is small; consider a multi-stage build so the
final image carries the toolchain + service without build debris.

### Darwin headers (macOS BOF compilation)

macOS BOFs compile as ELF but include Darwin SDK headers
(`BeaconFile.h`, plus whatever system headers the BOF uses). The full
macOS SDK cannot be redistributed. Plan:

- Vendor a **subset of APSL-licensed Darwin SDK headers** (the Apple
  Public Source License permits redistribution with compliance
  requirements - include the license text and a `LICENSE.darwin-headers`
  file documenting the source version).
- Reference implementation: the consumer repository vendors a text-stub
  subset for LINKING (see `LOCAL_NOTES.md` for its location); the BOF
  compile needs real HEADERS (declaration subset: kernel framework +
  libSystem headers as needed). Start minimal: the headers the curated
  BOF library actually uses, grow as needed.

### BOF library

Curate a set of ready-to-compile BOF sources so operators can compile
known-good tools without writing code:

- The **Dark-Agent BOF library** (the Dark-Agent project's BOF
  collection - the source of the ELF BOF dialect) - license-check and
  vendor or reference appropriately.
- A curated subset of the **Sliver Armory** BOF list - again license
  check; Armory BOFs are typically MIT/BSD but verify per item.
- Cobalt Strike's own example BOFs are NOT redistributable - do not
  vendor them.

Ship the library as source in the image (e.g. `/opt/boflib/`) and expose
a listing endpoint so a C2 UI can present a browsable catalog later.

## Architecture plan

1. **FastAPI** service (Python 3.12+) with uvicorn, behind **nginx** as
   the outer reverse proxy serving the sub-path. Rationale for the
   sub-path: the deployment host already serves other web services under
   one nginx with TLS; the new service joins as `location /bofcompiler/`.
2. Compilation runs in a work directory per request (tempdir, cleaned on
   completion). Bound concurrent compilations (semaphore) and cap request
   size (e.g. 2 MB source archives) and compile timeout (e.g. 60s).
3. The compile target selection maps to `clang --target ...` /
   `zig cc -target ...` per the matrix above; run the compiler in a
   restricted environment (no network; the toolchain needs none).
4. Statelessness: the service holds no state between requests beyond the
   token. Logging to stdout (container-style).
5. Packaging: Dockerfile + docker-compose example; publish multi-arch
   (amd64/arm64) images to ghcr the same way the base image repo does
   (see the obf-build-base repository's publish workflow for the proven
   pattern - matrix of native runners + manifest merge).
6. Wire-up into the consumer agent (separate work, done in the consumer
   repo once the service API stabilizes): the agent's `coff` command
   grows a "compile from source" flow that POSTs to the service and
   registers the returned object through its file upload mechanism,
   reusing the existing object-file task path. Do NOT implement this in
   this repo; this repo ships the service only. But keep the API
   operator-friendly for that flow (stable JSON errors, build log in the
   response).

## Testing expectations

- Unit tests for target/dialect mapping and the request validation.
- Integration test: compile a known trivial BOF per dialect/arch and
  assert the object parses (COFF: machine type + at least one symbol;
  ELF: ET_REL + correct machine).
- If the consumer checkout is available locally (see `LOCAL_NOTES.md`),
  end-to-end verification is possible: build the agent with an object
  produced by this service and run it through the coff loader's unit
  tests (the agent tree has a coff test suite in `src/coff/testkit.rs`).
- Obfuscation smoke test: compile with each level, `strings` the object,
  confirm sensitive strings are gone at light+.
- Reproducibility: identical input + fixed obf seed should produce
  identical objects (the toolchain supports a seed); expose an optional
  seed parameter.

## Environment / deployment notes

Deployment specifics (host, paths, reverse proxy configuration) live in
`LOCAL_NOTES.md` and must not enter the repository. Architectural rules
that DO belong here:

- The service deploys as a standalone `docker compose` project, separate
  from the C2 stack's own compose tree, so stack upgrades never touch
  it.
- TLS termination is the outer nginx's job; the service serves plain
  HTTP on localhost only.

## Known upstream facts (do not re-derive)

- The obfuscation passes are only in the forked toolchain; stock clang
  will silently ignore `-mllvm -sobf` etc. - always compile with
  `$OBF_CLANG`, never a system clang, when obfuscation is requested
  (better: reject obfuscation requests if the toolchain is absent).
- Object files must be **relocatable** (`.o` / COFF `/c`), never linked
  executables. The loaders link manually.
- The Dark-Agent dialect's `coffee` entrypoint receives `argc` in one
  register and `argv` in another with no standard prologue - compile
  with `-fPIC` and no unwind tables where possible to keep objects
  loader-friendly.
