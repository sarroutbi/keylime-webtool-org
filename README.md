# Keylime Monitoring Dashboard

A web-based security operations platform for centralized monitoring, management, and compliance of [Keylime](https://keylime.dev/) remote attestation infrastructure. The dashboard transforms Keylime from a CLI-driven security tool into a visual operations platform, reducing mean time to detect (MTTD) attestation failures from hours to seconds, centralizing policy and certificate lifecycle management, and providing tamper-evident audit trails for compliance reporting.

## Target Audience

- **Security Operations (SecOps) teams** -- real-time fleet attestation monitoring and alert triage
- **System Administrators** -- agent lifecycle management, performance monitoring, capacity planning
- **Compliance Officers** -- framework-mapped reports (NIST, PCI DSS, SOC 2, FedRAMP, CIS Controls)
- **DevSecOps Engineers** -- policy authoring, integration pipelines, CI/CD gating

## Architecture

```
                         +-------------------+
                         |    Browser (SPA)  |
                         |  React 18 + TS    |
                         |  Vite, Zustand,   |
                         |  TanStack Query   |
                         +--------+----------+
                                  |
                        TLS 1.3   |   WebSocket
                       (HTTPS)    |   (wss://)
                                  |
                         +--------v----------+
                         |   Rust Backend    |
                         |   Axum + Tokio    |
                         |   JWT / RBAC      |
                         +--+-----+------+--+
                            |     |      |
               mTLS (rustls)|     |      | mTLS (rustls)
                            |     |      |
                   +--------v-+ +-v----+ +v-----------+
                   | Keylime  | |TimescaleDB| | Redis  |
                   | Verifier | |(time-series)| |(cache) |
                   | (v2/v3)  | |           | |        |
                   +----------+ +-----------+ +--------+
                   | Keylime  |
                   | Registrar|
                   +----------+
```

The backend consumes Keylime's existing Verifier and Registrar REST APIs (v2 pull-mode and v3 push-mode) via mTLS **without requiring any modification to Keylime components**. It acts as a read-through cache and analytics aggregator, presenting a unified REST + WebSocket API to the frontend SPA.

## Repositories

This organization contains four repositories:

### [keylime-webtool-frontend](https://github.com/keylime-webtool/frontend)

React 18 + TypeScript single-page application providing the dashboard UI.

| | |
|---|---|
| **Stack** | React 18, TypeScript 5.6, Vite 6, React Router v6, Zustand 5, TanStack Query 5, Axios, Recharts |
| **Styling** | CSS Modules |
| **Testing** | Vitest + React Testing Library |
| **License** | Apache-2.0 |

**10 navigation modules:**

| Module | Route | Purpose |
|--------|-------|---------|
| Dashboard | `/` | Fleet KPIs, agent state distribution pie chart, recent alerts |
| Agents | `/agents`, `/agents/:id` | Agent list with search/filter/sort/bulk actions; 6-tab detail view (Timeline, PCR, IMA Log, Boot Log, Certificates, Raw Data) |
| Attestations | `/attestations` | Failure analytics, latency distribution, root cause suggestions |
| Policies | `/policies` | Unified IMA/MB policy list, versioning, two-person approval workflow |
| Certificates | `/certificates` | Certificate lifecycle, expiry tracking with tiered warnings |
| Alerts | `/alerts` | Alert lifecycle (New, Acknowledged, Investigating, Resolved) |
| Performance | `/performance` | Verifier cluster metrics, DB pool, circuit breaker status |
| Audit Log | `/audit` | Tamper-evident security event log, hash chain verification |
| Integrations | `/integrations` | Backend service connectivity status |
| Settings | `/settings` | Configuration, compliance framework reports |

**Key features:** global search (UUID/IP/hostname), time range selector (1h - 30d), WebSocket real-time updates with exponential backoff reconnection, role-aware UI rendering (Viewer/Operator/Admin).

### [keylime-webtool-backend](https://github.com/keylime-webtool/backend)

Rust async backend built on Axum and Tokio, serving the REST API and WebSocket endpoint.

| | |
|---|---|
| **Stack** | Rust 2021 edition, Axum, Tokio, sqlx (TimescaleDB), redis, reqwest + rustls, jsonwebtoken, openidconnect |
| **Observability** | tracing + tracing-subscriber, OpenTelemetry, Prometheus metrics |
| **Testing** | Mockoon integration tests simulating Keylime Verifier/Registrar |
| **License** | Apache-2.0 |

**64 API endpoints** across 11 handler modules:

| Module | Endpoints | Purpose |
|--------|-----------|---------|
| agents | 11 | Fleet list, search, detail, PCR values, IMA log, boot log, certificates, raw data, bulk actions |
| attestations | 10 | Summary, failures, incidents, pipeline visualization, push/pull mode analytics, state machine |
| policies | 11 | CRUD, versioning, diff, rollback, impact analysis, assignment matrix, two-person approval |
| certificates | 4 | List, detail, expiry summary, renewal |
| alerts | 7 | List, acknowledge, investigate, resolve, dismiss, thresholds, notifications |
| audit | 3 | Event list, hash chain verification, export |
| kpis | 1 | Real-time fleet KPIs |
| compliance | 3 | Framework listing, report generation, export |
| performance | 5 | Verifier metrics, database stats, API response times, config drift, capacity planning |
| integrations | 4 | Verifier/Registrar health, SIEM, revocation channels, durable backends |
| auth | 4 | OIDC login/callback, token refresh, logout |

**WebSocket:** `/ws/events` for real-time push updates (KPIs, agent state changes, alerts).

**Keylime integration:** Circuit breaker pattern (threshold: 5 failures, reset: 60s), concurrent log fetch semaphore (max 5 parallel requests), dual API version support (v2 pull + v3 push).

**`#![forbid(unsafe_code)]`** enforced at the crate root.

### [keylime-webtool-doc](https://github.com/keylime-webtool/doc)

Project documentation, specifications, and presentation materials.

| | |
|---|---|
| **Build** | GNU Make + pdflatex (Beamer) |
| **License** | CC BY-SA 4.0 |

**Contents:**

| Path | Description |
|------|-------------|
| `spec/SRS-Keylime-Monitoring-Tool.md` | Software Requirements Specification -- 70 functional, 23 non-functional, and 29 security requirements with Gherkin acceptance criteria. Includes implementation refinements (Section 7) tracking data models, API contracts, and enumerations. |
| `spec/SDD-Audit-Report.md` | SDD audit review assessing sprint-readiness, INVEST criteria compliance, and Gherkin quality. |
| `slides/20260226-Keylime-Monitoring-Tool/` | Technical presentation (Beamer/LaTeX) -- architecture, UI components, verification pipeline, security model. |
| `slides/20260305-Keylime-Monitoring-Tool-Stakeholders/` | Stakeholder-oriented presentation -- problem statement, solution overview, roadmap, benefits. |
| `icons/` | Project branding assets. |

### [keylime-webtool-org](https://github.com/keylime-webtool/.github) (this repository)

Organization-level profile and governance documentation.

## Tech Stack Summary

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | React 18 + TypeScript | Component-based SPA with type safety |
| Build | Vite 6 | Fast HMR dev server, optimized production builds |
| Routing | React Router v6 | Client-side routing with protected routes |
| Client State | Zustand 5 | Auth store (JWT, user role, permissions) |
| Server State | TanStack Query 5 | Caching, polling, mutation with automatic invalidation |
| HTTP | Axios | JWT interceptor, response envelope unwrapping, 401 redirect |
| Charts | Recharts | Agent state distribution, attestation analytics |
| Backend | Rust (Axum + Tokio) | Async HTTP/WebSocket server |
| Database | TimescaleDB (PostgreSQL) | Time-series attestation history and metrics |
| Cache | Redis | Tiered TTLs (agent list 10s, detail 30s, policies 60s, certs 300s) |
| TLS | rustls + tokio-rustls | TLS 1.3 browser-facing, mTLS to Keylime APIs |
| Auth | OIDC/SAML + JWT | 15-min token expiry, refresh rotation, server-side revocation |
| Docs | Beamer/LaTeX | Presentation slides with Red Hat theme |
| CI | GitHub Actions | Build validation, markdown linting, Fedora container matrix |

## Security Model

### Authentication and Authorization

- **Identity:** OIDC/SAML via external identity provider (no local passwords)
- **Sessions:** Short-lived JWT (15-minute expiry) with refresh token rotation and replay detection
- **MFA:** Mandatory for Admin role (enforced at IdP, verified via token claim)
- **RBAC:** Three-tier model enforced at the backend proxy layer

| Capability | Viewer | Operator | Admin |
|------------|--------|----------|-------|
| View fleet, analytics, policies, audit log | Yes | Yes | Yes |
| Export reports (CSV/PDF) | -- | Yes | Yes |
| Acknowledge/manage alerts | -- | Yes | Yes |
| Reactivate/stop agents | -- | Yes | Yes |
| Create/edit/delete policies | -- | -- | Yes |
| Delete agents from verifier | -- | -- | Yes |
| Configure alert thresholds | -- | -- | Yes |

### Transport Security

- **Browser to Dashboard:** TLS 1.3 minimum
- **Dashboard to Keylime:** mTLS with rustls (private key via HSM/PKCS#11 or HashiCorp Vault -- never cleartext on disk)
- **Dashboard to Database/Cache:** TLS encrypted

### Data Protection

- Raw TPM quotes, IMA logs, boot logs, and PoP tokens are **never cached, stored, or logged** -- pass-through only
- Cache entries are signed (HMAC) with TTLs to mitigate poisoning
- Audit log is hash-chained (SHA-256) with RFC 3161 timestamp anchoring and Rekor transparency log checkpoints
- Minimum 1-year audit log retention for compliance

### Policy Governance

- **Two-person rule:** Policy changes require drafter + separate approver (N-of-M quorum configurable)
- **Time-limited approval window:** Default 24 hours, configurable
- **Emergency bypass:** Break-glass mechanism with mandatory justification and CRITICAL audit trail
- **Impact analysis:** Pre-update analysis categorizing agents as unaffected/affected/will-fail

## Getting Started

### Prerequisites

- **Node.js 18+** and **npm** (frontend)
- **Rust 1.75+** with cargo (backend)
- **Keylime** Verifier and Registrar (or Mockoon for mock testing)
- Optionally: TimescaleDB (PostgreSQL), Redis

### Quick Start with Mocks

```bash
# 1. Start mock Keylime services (requires mockoon-cli: npm install -g @mockoon/cli)
cd keylime-webtool-backend
mockoon-cli start --data test-data/verifier.json --port 3000 &
mockoon-cli start --data test-data/registrar.json --port 3001 &

# 2. Start the backend (default: http://localhost:8080)
RUST_LOG=info cargo run

# 3. In a separate terminal, start the frontend (default: http://localhost:5173)
cd keylime-webtool-frontend
npm install
npm run dev
```

The frontend dev server proxies `/api/*` and `/ws` requests to the backend at `localhost:8080`.

### Environment Variables

**Frontend** (prefix `VITE_` for Vite exposure):

| Variable | Default | Purpose |
|----------|---------|---------|
| `VITE_API_BASE_URL` | `http://localhost:8080` | Backend API root URL |
| `VITE_WS_URL` | `ws://localhost:8080/ws` | WebSocket endpoint for real-time updates |

**Backend:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `KEYLIME_VERIFIER_URL` | `http://localhost:3000` | Keylime Verifier API base URL |
| `KEYLIME_REGISTRAR_URL` | `http://localhost:3001` | Keylime Registrar API base URL |
| `RUST_LOG` | (none) | Tracing log level filter (e.g., `info`, `debug`) |

### Running Tests

```bash
# Frontend
cd keylime-webtool-frontend
npm run test          # watch mode
npm run test -- --run # single run
npm run lint          # ESLint

# Backend (with Mockoon mocks)
cd keylime-webtool-backend
bash tests/mockoon_tests.sh

# Documentation (build presentations)
cd keylime-webtool-doc
make -C slides test
```

## Deployment

The system is designed for **air-gapped environments** with no runtime internet access:

- All frontend assets are self-contained (no CDN)
- Backend compiles to a single binary with embedded static assets
- Fonts, icons, and scripts are bundled
- Offline EK certificate validation via pre-loaded TPM vendor CA certificates
- Update packages are GPG-signed with SBOM (SPDX/CycloneDX)

**Supported deployment methods:**

| Method | Description |
|--------|-------------|
| OCI Container | Pre-built container images with all layers |
| Kubernetes | Helm chart for backend + frontend + database |
| RPM | System package with systemd service unit |
| systemd | Direct binary with systemd management |

**High Availability:**

- Active/Passive: < 30s RTO, 0 RPO for committed transactions
- Active/Active: For 5,000+ agent deployments with load distribution

## Requirements Coverage

The project is specified using **Spec-Driven Development (SDD)** methodology. The full specification lives in [`keylime-webtool-doc/spec/SRS-Keylime-Monitoring-Tool.md`](https://github.com/keylime-webtool/doc/blob/main/spec/SRS-Keylime-Monitoring-Tool.md).

| Category | Count | Examples |
|----------|-------|---------|
| **Functional Requirements** | 70 | Fleet KPIs, agent management, attestation analytics, policy CRUD with two-person approval, certificate lifecycle, alert workflow, audit logging, compliance reports, SIEM integration |
| **Non-Functional Requirements** | 23 | 30s KPI refresh, 10K WebSocket connections, 100K+ agent scale, air-gapped deployment, WCAG 2.1 AA accessibility, circuit breaker, rate limiting, cache TTLs |
| **Security Requirements** | 29 | OIDC/SAML auth, MFA for Admin, mTLS with HSM keys, TLS 1.3, hash-chained audit log, SSRF protection, data classification, `#![forbid(unsafe_code)]` |

### Implementation Phasing

**Phase 1 -- Secure Foundation:** Agent fleet monitoring, OIDC/SAML authentication, three-tier RBAC, tamper-evident audit log, mTLS to Keylime APIs.

**Phase 2 -- Operations:** Attestation analytics, policy management, certificate monitoring, alert notifications, push mode (v3) support, SIEM integration, WebSocket real-time updates.

**Phase 3 -- Enterprise Scale:** Multi-tenancy, compliance reports, incident response integration (ServiceNow, Jira, PagerDuty), HA deployment, air-gapped packaging, WCAG 2.1 AA accessibility.

## Compliance Frameworks

The dashboard maps attestation capabilities to specific compliance controls:

| Framework | Mapped Controls |
|-----------|----------------|
| **NIST SP 800-155** | BIOS integrity measurement |
| **NIST SP 800-193** | Platform firmware resilience |
| **PCI DSS 4.0** | Req 11.5 (file integrity monitoring), Req 10.2 (audit trail) |
| **SOC 2 Type II** | CC7.1 (monitoring activities), CC6.1 (logical access controls) |
| **FedRAMP** | CA-7 (continuous monitoring), SI-7 (software integrity) |
| **CIS Controls v8** | 2.5 (allowlisted software) |

## License

| Repository | License |
|-----------|---------|
| keylime-webtool-frontend | [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0) |
| keylime-webtool-backend | [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0) |
| keylime-webtool-doc | [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) |
