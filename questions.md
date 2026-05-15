# Agent Builder — Implementation Questions

These questions cover everything I need resolved before planning the implementation.
Grouped by area. Please answer directly under each question.

---

## 1. Scope and Starting Point

**Q1.1 — What does "complete the project" mean right now?**
The design docs recommend a specific build sequence:
1. `agent-init` — the container PID 1 binary (~200 lines, fully specified)
2. Agent SDK skeleton (`agent-sdk` library crate) — wraps the wire protocol, session tokens, typed calls
3. `agent-router` binary — the intra-container message bus
4. Required module adapters (pipeline, tracer, permissions, context, sandbox)

Are you asking me to work through this full sequence, or to start somewhere specific?

> Answer:

---

**Q1.2 — Where do the binaries live in the repo?**
The design docs describe 6 submodules but the Cargo workspace only has 5 members (no `build_system_standards`). The current layout also has a root `src/main.rs` stub that isn't in the spec. How should I map the binaries to crates?

Proposed mapping (confirm or correct):
- `agent_instance` → `agent-router` binary + `agent-sdk` library crate
- `agent_container` → `agent-init` binary + all module adapter binaries
- `agent_controller` → `agent-controller` binary
- `agent_builder` → lifecycle manager (`agent-mlm`) binary
- `agent_design_compiler` → build tool (`agent-build`) binary
- Root `src/main.rs` → delete (not in spec)

> Answer:

---

**Q1.3 — Is `build_system_standards` a Rust crate or spec-only?**
The design docs list it as a Cargo workspace member (a "spec + schema validation library"), but it has no `Cargo.toml` and isn't in the workspace yet. Should it become a real crate — a shared library of types and validation logic used by the build tool and router?

> Answer:

---

## 2. Naming Conventions

**Q2.1 — What should the agent module format be called?**
The design docs use "PAF" (Panorama Agent Format) for the module file format, build artifacts, and wire protocol. Since the builder is generic, do you want a new short name (e.g., "AMF", "ABF", or something else), or should I just use descriptive names throughout (e.g., "agent-build", "agent-format", "agent manifest") without an acronym?

> Answer:

---

**Q2.2 — What should the intra-container wire protocol be called?**
The design docs call it "PMP" (PAF Message Protocol). Same question — new acronym, or descriptive?

> Answer:

---

**Q2.3 — What should the controller↔adapter protocol be called?**
Currently "CAP" (Controller Adapter Protocol). Keep or rename?

> Answer:

---

## 3. Rust Ecosystem Choices

The specs define the *what* but not the *how* for Rust dependencies. These choices affect every crate.

**Q3.1 — Async runtime?**
The router, controller, and lifecycle manager all need async I/O. Which runtime?
- **tokio** (most common, best ecosystem fit for epoll/kqueue)
- **async-std**
- Other

> Answer:

---

**Q3.2 — TOML parsing for the build tool?**
Stage 1 (Parser) reads TOML into an AST. Options:
- `toml` crate (standard, serde-based, loses span info for error reporting)
- `toml_edit` (preserves spans and formatting — needed for Stage 3 Formatter and accurate error locations)

Stage 3 normalizes and rewrites files in place, which strongly favors `toml_edit`. Confirm?

> Answer:

---

**Q3.3 — Serialization format for binary build artifacts?**
The build tool produces `router-dispatch.bin` and `permission-matrix.bin`, which the router reads at startup. Options:
- `bincode` (fast, compact, Rust-native)
- `postcard` (no-std friendly)
- `flatbuffers` / `capnproto` (zero-copy, but complex)
- MessagePack

> Answer:

---

**Q3.4 — Runtime payload validation?**
Request/response payloads are validated against JSON Schemas. The router can validate each incoming call payload. Options:
- `jsonschema` crate
- `valico`
- Skip runtime validation in v1 (validate only at build time)

> Answer:

---

**Q3.5 — HTTP client for the pipeline adapter?**
The pipeline module adapter makes HTTPS calls to a model API with SSE streaming. Options:
- `reqwest` (most common, tokio-native, SSE via `eventsource-client`)
- `hyper` directly (lower-level)

> Answer:

---

**Q3.6 — Ed25519 signing for the sealed manifest?**
The build tool signs the container manifest at Stage 11. Options:
- `ed25519-dalek`
- `ring`

> Answer:

---

## 4. Design Gaps (Unspecified Areas I Will Encounter)

Several items are explicitly marked "not yet designed" in the docs. I need to know whether to design them, skip them, or wait.

**Q4.1 — Custom KV backend gRPC proto**
The `kv` module supports a built-in KV backend with a gRPC interface, but the proto file doesn't exist yet. Should I:
- Design the proto file as part of this pass
- Skip the custom backend and implement only SQLite and Redis for now
- Other

> Answer:

---

**Q4.2 — Internal search adapter interface**
The `search` module supports an internal search index backend, but its interface isn't specified. Should I implement only the external search backends (Brave, Serper, Exa) and leave internal as a stub?

> Answer:

---

**Q4.3 — Module Registry**
The design docs flag this as a "large" item blocking the lifecycle manager and binary deployment. It needs: an API spec, content-addressing, Ed25519 signing, publishing workflow. Should I:
- Design and implement it as part of this pass
- Stub it out (hardcode binary paths, skip the registry entirely for now)
- Other

> Answer:

---

**Q4.4 — Dockerfile generator**
The lifecycle manager is supposed to generate Dockerfiles from the sealed agent manifest. The hand-authored `Dockerfile` in `design-docs/` is the template. Should I:
- Implement the generator (template engine → Dockerfile output)
- Use the hand-authored Dockerfile as-is for now
- Other

> Answer:

---

**Q4.5 — Binary signing and verification**
Ed25519 trust root, key rotation, revocation — not yet fully designed. For now, should signing be:
- Always required (fail if no key provided)
- Optional (`--no-sign` flag, warn but continue)
- Skip for now (generate manifests unsigned)

> Answer:

---

## 5. Repository Structure

**Q5.1 — Multiple binaries per crate?**
Some crates will contain multiple binaries. `agent_container` should hold `agent-init` plus ~25 module adapter binaries. Cargo supports this via `[[bin]]` entries. Is this the intended structure, or should each binary live in its own crate?

> Answer:

---

**Q5.2 — Shared types crate?**
Several binaries share types: wire protocol frame types, manifest schema structs, capability definitions. Should there be a shared `agent-core` crate, or should shared types live in `build_system_standards` (if it becomes a crate)?

> Answer:

---

**Q5.3 — `agent_instance` is currently a library, not a binary.****
The router should be a binary. The SDK should be a library. Should `agent_instance` become a workspace with both, or a single crate with both `[lib]` and `[[bin]]` targets?

> Answer:

---

## 6. Testing Strategy

**Q6.1 — How to test the router without a full container?**
The router reads binary artifacts produced by the build tool. For unit testing in isolation, should I:
- Write test helpers that construct these artifacts programmatically
- Run the build tool as a test fixture to generate real artifacts
- Use a test-only mode that bypasses binary artifact loading

> Answer:

---

**Q6.2 — Integration testing approach?**
A full integration test would require Docker, multiple processes, and real sockets. For now:
- Write integration tests that spawn actual processes locally (no Docker)
- Write only unit tests for each component in isolation
- Both

> Answer:

---

**Q6.3 — Coverage expectation?**
E.g., "focus on critical path only", "aim for 80%", "just make sure it compiles and the happy path works."

> Answer:

---

## 7. Operational Environment

**Q7.1 — Primary target platform?**
The file watcher uses inotify (Linux), FSEvents (macOS), or polling fallback. Is the primary deployment target:
- Linux only (Docker containers)
- macOS for development + Linux for deployment
- Both equally

> Answer:

---

**Q7.2 — Minimum Rust version?**
The workspace currently uses `edition = "2024"` (requires Rust 1.85+). Is "latest stable" fine, or is there a minimum toolchain to target?

> Answer:

---

**Q7.3 — Where do the Docker packaging files belong?**
`Dockerfile`, `docker-compose.yml`, `paf-build.service`, and `paf-build-wrapper.sh` are currently in `build_system_standards/design-docs/`. Where should they live in the final layout?
- Move to `agent_design_compiler/` alongside the build tool binary
- Leave in `build_system_standards/` as reference artifacts
- Other

> Answer:

---

## 8. Priority and Order

**Q8.1 — What is the single most important deliverable right now?**
This determines whether to go top-down (build tool first, so manifests drive everything else) or bottom-up (init + router first, so the runtime is exercisable before the build system exists).

> Answer:

---

**Q8.2 — Does the build tool need to exist before the router can be tested?**
The router reads binary artifacts produced by the build tool. If the router comes first, I need either a build-tool stub or hand-crafted test fixtures. Is that acceptable?

> Answer:

---

**Q8.3 — Controller priority?**
The controller is the container's only external face and required for production operation. For early development, can the container start without a connected controller (i.e., no constraint enforcement but no blocked startup), or must the controller be functional before any end-to-end test is meaningful?

> Answer:

---
