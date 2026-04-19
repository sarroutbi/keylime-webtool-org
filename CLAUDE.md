# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This is a multi-repo workspace for the **Keylime Monitoring Dashboard** -- a security operations platform providing centralized monitoring, management, and compliance for [Keylime](https://keylime.dev/) remote attestation infrastructure. Each subdirectory is an independent git repository.

| Repository | Language | Role |
|------------|----------|------|
| `keylime-webtool-backend` | Rust (Axum) | REST + WebSocket API server; consumes Keylime APIs via mTLS |
| `keylime-webtool-frontend` | TypeScript (React 18 + Vite) | SPA dashboard |
| `keylime-webtool-doc` | LaTeX/Markdown | SRS specification, architecture docs, presentations |
| `keylime-webtool-org` | Markdown | GitHub organization profile and governance |
| `keylime` | Python | Upstream Keylime: Verifier, Registrar, Tenant CLI |
| `rust-keylime` | Rust (Actix-web) | Upstream Keylime agent (pull + push modes) |

The dashboard repos (`-backend`, `-frontend`) are the primary development targets. `keylime` and `rust-keylime` are upstream dependencies consumed via their REST APIs -- read/reference only.

## Architecture

```
Browser (React SPA :5173)
  --TLS 1.3-->  keylime-webtool-backend (:8080)
                    |         |         |
                  mTLS       TCP       TCP
                    |         |         |
              Keylime APIs  TimescaleDB  Redis
              (:3000/:3001)  (time-series) (cache)
```

The backend is a pass-through aggregation layer: it calls Keylime Verifier (v2 pull-mode) and Registrar (v3 push-mode) via mTLS, caches responses in Redis with tiered TTLs, stores attestation history in TimescaleDB, and pushes real-time updates to the frontend over WebSocket.

## Sub-Project CLAUDE.md Files

Both active development repos have their own detailed CLAUDE.md with tech stack, architecture, commands, and constraints:
- `keylime-webtool-backend/CLAUDE.md`
- `keylime-webtool-frontend/CLAUDE.md`

Refer to those for sub-project-specific guidance. The sections below provide cross-cutting context.

## Build & Test Commands

### Backend (keylime-webtool-backend)

```bash
cargo build                    # dev build
cargo test                     # unit tests
cargo test <name>              # single test
cargo clippy -- -D warnings    # lint
cargo fmt -- --check           # format check
bash tests/mockoon_tests.sh    # integration tests (auto setup/teardown)
bash scripts/pre-commit.sh     # all CI checks locally
```

Mockoon integration tests simulate Keylime APIs using mock data in `test-data/`. Tests are gated behind the `mockoon` feature flag and env vars `MOCKOON_VERIFIER`/`MOCKOON_REGISTRAR`.

### Frontend (keylime-webtool-frontend)

```bash
npm install                    # install dependencies
npm run dev                    # dev server (:5173, proxies /api/* to :8080)
npm run build                  # production build
npm run lint                   # ESLint (zero warnings)
npm run test                   # Vitest watch mode
npm run test -- --run          # single test run
npx tsc -b                     # type check
bash scripts/pre-commit.sh     # all CI checks locally
```

### Upstream Keylime (keylime -- Python)

```bash
make check                     # all tox lint environments (pylint, mypy, black, isort, pyright)
tox -e py3                     # unit tests
```

### Upstream Agent (rust-keylime)

```bash
make                           # build all binaries (debug)
make RELEASE=1                 # release build
cargo test                     # unit tests (non-TPM)
```

### Documentation (keylime-webtool-doc)

```bash
cd slides && make              # build all LaTeX presentations
make test                      # draft-mode compilation check
```

## CI Workflows

All repos enforce GPG-signed commits. Key workflow patterns:

- **Backend:** `cargo fmt` + `clippy` + `cargo test` + Mockoon integration + `cargo audit` + `cargo machete` + `shellcheck`
- **Frontend:** `tsc -b` + ESLint + Vitest + Vite build + `npm audit`
- **rust-keylime:** Format check + panic detection + Fedora container tests with swtpm + tarpaulin coverage
- **keylime (Python):** pre-commit style checks + TPM tests in Fedora container + pylint/mypy/pyright

CI workflows use path filtering to skip unrelated jobs.

## Key Specifications

The full SRS lives at `keylime-webtool-doc/spec/SRS-Keylime-Monitoring-Tool.md` (70 FRs, 23 NFRs, 29 SRs with Gherkin acceptance criteria). Requirements are tagged (e.g., NFR-005, SR-023) and referenced in code and CLAUDE.md files.

## Cross-Cutting Constraints

- **`#![forbid(unsafe_code)]`** enforced on the backend crate (SR-023)
- **Air-gapped deployment** -- no external CDN or runtime internet access; all assets self-contained
- **Sensitive data pass-through only** -- raw TPM quotes, IMA logs, boot logs, PoP tokens are never cached, stored, or logged (SR-013/014)
- **Two-person policy approval** -- policy drafter cannot be the approver
- **RBAC three-tier model** -- Viewer (read-only), Operator (read + write), Admin (full + MFA required)
- **TLS 1.3 minimum** for browser connections; TLS 1.2+ for Keylime API communication

## License

- Code repositories: Apache-2.0
- Documentation: CC BY-SA 4.0
