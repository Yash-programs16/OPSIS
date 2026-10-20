OPSIS — Server Observability and Safe Remediation Appliance (Mode B First)
Project status: architecture and scaffold. This README describes the intended product and canonical repository layout. File names and proposed functions are implementation plans unless code has been written and tested.

OPSIS is a local-first server observability and reliability appliance for indie developers. The flagship product path is Mode B: a lightweight Go agent on the user's Linux server (nginx first) collects real host and request telemetry, and an Arduino UNO Q appliance acts as the always-on control plane, local store, detection engine, and dashboard host. OPSIS correlates CPU, memory, swap, disk, process/restart state, per-route request rates, HTTP error ratios, p50/p95/p99 latency, and error-log patterns. It turns that evidence into incidents, ranked likely causes, and recommendations. Approval-gated remediation is introduced only after telemetry and verification are trustworthy.

External URL probes are supporting evidence and a fallback for users who cannot install the agent; they are not the central product identity. An application health endpoint is another extension. OPSIS aims to provide a simple server observability and reliability workflow without forcing users to assemble and operate a full Prometheus/Grafana stack. Monitoring and detection must work locally without cloud availability. Optional Neon Postgres synchronization is late-stage and must never be on the detection critical path.
Contents
1. Product behavior and monitoring modes
2. Architecture and trust boundaries
3. Final repository layout
4. File-by-file responsibilities and proposed functions
5. End-to-end data flow
6. API and RPC contracts
7. Local database design and retention
8. Detection, diagnosis, and remediation rules
9. Security and privacy
10. Technology stack
11. Development setup and commands
12. Chronological implementation plan
13. Testing and demo scenarios
14. Team boundaries
15. Definition of done
1. Product behavior and monitoring modes
Flagship user journey — server observability first
1. Power on the UNO Q appliance and connect network, power, and (optionally) a display/USB-C hub.
2. Complete first-boot setup: network, time zone, and local administrator account.
3. Create a project and register a supported Linux server. The first supported server path is Linux with nginx.
4. Install the Go opsis-agent on that server using the documented installer, then complete one-time secure pairing with the appliance.
5. The server agent samples host metrics about every 5 seconds, follows/normalizes nginx access and error logs, and calculates bounded per-route summaries in roughly 10-second windows.
6. The board receives summaries over an outbound, mutually authenticated gRPC stream, validates metric names/units/timestamps, and batches them into local SQLite storage.
7. The dashboard shows CPU, memory, swap, disk, process/restart state, request rate, 4xx/5xx ratios, p50/p95/p99 latency, and log patterns. The user can inject a controlled test fault and inspect the resulting evidence.
8. OPSIS uses server-first rules and baselines to open incidents, presents evidence-backed cause hypotheses, and sends alerts. ML is added later in shadow mode.
9. In a later phase, the user can approve a specific allow-listed action. OPSIS records it, applies an expiry on the server, independently checks recovery, and reverts/escalates if verification fails.
10. External URL probes can be configured to cross-check the server evidence or to provide a fallback when the user's hosting environment does not permit agent installation.
Monitoring modes (ordered by product priority)
Mode	Setup	What OPSIS can observe	Product role and limits
B — Server agent (flagship)	Install the Go agent on a supported Linux server; nginx is the first target.	Host CPU/load, memory/swap, disk/inodes, file descriptors, configured process RSS/restarts, nginx per-route request rate, 4xx/5xx, bytes, p50/p95/p99 and error-log patterns. Later, controlled actions.	Core product path and first end-to-end demo. Agent permissions and log formats must be explicitly configured and tested.
A — External probes (supporting)	Register a public URL and routes; no software on the monitored server required.	HTTP status, DNS, TCP connect, TLS, first-byte/total latency, response size, keyword checks and certificate expiry.	Fallback for managed hosts and independent cross-check for server-side evidence. A URL alone cannot reveal private CPU, memory, logs or real request counts.
C — Application health endpoint (extension)	Add an app-owned OPSIS health/telemetry endpoint.	Mode A plus application-reported dependency and database timings or other explicitly exposed metrics.	Improves diagnosis when agent access is unavailable; does not grant server access or enable server actions.


Capability honesty: a metric is shown as available only when the required data source exists and is healthy. Unsupported or missing measurements must be marked unavailable—with a reason and the setup needed to enable them—never fabricated as zero.
Automation modes
- Observe: collect and report; do not execute remediation.
- Recommend: provide evidence-backed suggestions; the user performs the change. This is the intended default.
- Approve-first: require approval of the exact action and parameters.
- Auto: permit only individually enabled, reviewed, allow-listed actions with bounded TTL, cooldown, rate limits, preconditions, independent verification, and a safe revert path. Disabled by default.
The Python model never executes commands. Its output is one input to Go's detection and policy logic, not authorization to change a system.
2. Architecture and trust boundaries
System diagram
The primary telemetry path is server agent → authenticated gRPC stream → board ingestion → local store and analysis → server dashboard. External probes add independent evidence; they do not replace server-level observability.
 USER'S LINUX SERVER                                      ARDUINO UNO Q 4 GB
 ┌────────────────────────────┐             outbound      ┌──────────────────────────────┐
 │ opsis-agent (Go)           │  mTLS gRPC  stream        │ opsisd (Go)                  │
 │ - host sampler (~5 s)      ├──────────────────────────>│ - pairing + ingestion        │
 │ - nginx log tail/parse     │                           │ - SQLite WAL + rollups       │
 │ - route aggregates (~10 s)│<──────────────────────────┤ - rules/baselines + incidents│
 │ - typed action registry    │ approved, typed commands  │ - RCA/policy/verification     │
 │ - persistent action TTL    │                           │ - REST + SSE API              │
 └────────────────────────────┘                           └──────────────┬───────────────┘
                                                                        │ REST + SSE
                                                                        v
                                                          React + TypeScript Console
                                                          - server/route telemetry
                                                          - evidence + incident timeline
                                                          - recommendations/action history

 Board-side supporting services:
   opsisd ── gRPC over Unix socket ──> opsis-ml (Python; shadow-mode inference)
   opsisd ── supporting probes ──────> public routes / DNS / TLS / HTTP phases
   opsisd ── local outbox ───────────> Neon Postgres (optional, last phase)
   opsisd ── optional bridge ────────> MCU LED / buzzer / acknowledge button
Key design decisions
- Mode B is the flagship. The first meaningful end-to-end demo is a real Linux/nginx server, the Go agent, secure telemetry ingestion, local persistence, and a live server/route dashboard.
- One root Go module is shared by cmd/opsisd, cmd/opsis-agent, cmd/opsisctl, internal/, and agent/.
- SQLite is the system of record on the board. Monitoring and detection do not depend on cloud availability.
- SQLite uses WAL, a busy timeout, batched writes, and bounded retention. Avoid writing each metric every second indefinitely.
- The ML service is a separate process communicating over a local Unix socket. If ML is unavailable, rules and Go baseline/forecast fallback continue.
- Browser communication is REST/JSON plus Server-Sent Events (SSE); browsers do not use native gRPC directly.
- The board initiates the gRPC stream to the user's server. This supports outbound connectivity through common NAT setups and avoids requiring an inbound home-network connection to the board.
- Agent pairing uses a one-time token and mutual TLS (mTLS) with pinned certificates. Revocation must be supported; telemetry messages need schema versions, sequence numbers, bounded sizes and defined reconnect behavior.
- The agent runs as a dedicated non-root user. Any privileged operation is a later feature and must use narrow allowlists; the agent never exposes a remote shell.
- Neon synchronization is optional, late-stage, and asynchronous through an outbox. Raw telemetry stays on the appliance by default.
Component responsibilities
Component	Runtime	Owns
opsisd	Go on the UNO Q	Public REST/SSE API, route probes, local persistence, agent connection, telemetry ingestion, aggregation, rule/baseline detection, ML client, incident lifecycle, diagnosis, policy, action orchestration, notifications, optional Neon synchronization, embedded web assets.
opsis-ml	Python on the UNO Q	Feature validation, anomaly scoring, time-to-exhaustion forecasting, log-template mining, model registry/training/evaluation. It has no privileged action interface.
opsis-agent	Go on the user's Linux server	Core telemetry producer: host sampling, nginx log collection/parsing, route statistics, mTLS gRPC server, and—only in a later phase—narrowly scoped actions that expire independently of the board.
opsisctl	Go CLI on the board	Status/diagnostics, support bundle, configuration validation, pairing helper, update/rollback, and factory reset commands.
web/	Browser	Setup wizard, projects, sites, routes, live metrics, incidents, evidence, actions, alerts, settings, and kiosk wall board. It only calls the Go API.
firmware/	Optional MCU firmware	LED/buzzer/button and heartbeat behavior. The product must continue to work without this optional component.


Key design decisions
- One root Go module is shared by cmd/opsisd, cmd/opsis-agent, cmd/opsisctl, internal/, and agent/.
- SQLite is the system of record on the board. Monitoring and detection do not depend on cloud availability.
- SQLite uses WAL, a busy timeout, batched writes, and bounded retention. Avoid writing each metric every second indefinitely.
- The ML service is a separate process communicating over a local Unix socket. If ML is unavailable, rules and Go baseline/forecast fallback continue.
- Browser communication is REST/JSON plus Server-Sent Events (SSE); browsers do not use native gRPC directly.
- The board initiates the gRPC stream to the user's server. This supports outbound connectivity through common NAT setups and avoids requiring an inbound home-network connection to the board.
- Agent pairing uses a one-time token and mutual TLS (mTLS) with pinned certificates. Revocation must be supported.
- Neon synchronization is optional, late-stage, and asynchronous through an outbox. Raw telemetry stays on the appliance by default.
Trust boundaries
1. Browser to opsisd: authenticated session, CSRF protection on mutations, bounded request bodies, and secure cookie settings.
2. Board to agent: mTLS and paired certificate validation; agent accepts typed protocol messages, not remote shell input.
3. Go to Python: local Unix socket, schema/version checks, deadlines, size limits, and fallback on service failure.
4. Target URLs to probe engine: hostile/untrusted input; SSRF protections must be applied on the initial URL and every redirect.
5. Policy to action executor: an anomaly or model score cannot skip authorization, allowlist, evidence freshness, preconditions, approval, cooldown, or TTL checks.
6. Board to Neon: outbound sync only; cloud outages must delay sync, not stop local monitoring.
3. Final repository layout
This is the canonical target layout from the final HLD/LLD. Names are intentional: cmd/, internal/, agent/, ml/, and web/ live at the repository root. The older backend-go/, ml-python/, frontend/, and root migrations/ skeleton paths are not parallel implementations; migrate any useful files into this canonical layout when practical.
opsis/
├── README.md
├── Makefile
├── go.mod
├── go.sum
├── buf.yaml
├── buf.gen.yaml
├── .gitignore
├── .editorconfig
├── .golangci.yml
├── opsis.code-workspace
├── .vscode/
│   ├── settings.json
│   ├── launch.json
│   ├── tasks.json
│   └── extensions.json
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
├── proto/
│   ├── agentlink/v1/agentlink.proto
│   └── ml/v1/ml.proto
├── gen/                         # Generated; never edit by hand
│   ├── go/
│   └── python/
├── cmd/
│   ├── opsisd/main.go
│   ├── opsis-agent/main.go
│   └── opsisctl/main.go
├── internal/                    # Board-side Go implementation
│   ├── config/config.go
│   ├── logx/log.go
│   ├── core/{models.go,errors.go}
│   ├── api/{server.go,routes.go,middleware.go,auth.go,sse.go,dto.go,handlers_*.go}
│   ├── store/{db.go,migrate.go,*_repo.go,retention.go}
│   │   └── migrations/001_core.sql ... 005_ml_models_outbox.sql
│   ├── probe/{scheduler.go,http.go,tls.go,dns.go,selfcheck.go,fingerprint.go,ssrf.go}
│   ├── agentlink/{client.go,stream.go,pairing.go,certs.go}
│   ├── ingest/{pipeline.go,normalise.go,redact.go}
│   ├── aggregate/{window.go,rollup.go,features.go}
│   ├── detect/{rules.go,baseline.go,forecast.go,hysteresis.go,fusion.go,coverage.go}
│   ├── mlclient/client.go
│   ├── rca/{layers.go,evidence.go,rank.go,explain.go}
│   ├── incident/{engine.go,state.go,dedupe.go}
│   ├── policy/{model.go,evaluate.go,guards.go}
│   ├── action/{executor.go,verify.go,revert.go}
│   ├── notify/{router.go,telegram.go,email.go,webhook.go}
│   ├── sysctl/{wifi.go,timesync.go,update.go,backup.go}
│   ├── neonsync/{outbox.go,worker.go,restore.go}
│   └── mcu/bridge.go
├── agent/                       # Go process installed on user's server
│   ├── config/config.go
│   ├── pairing/{server.go,certs.go}
│   ├── grpcserver/server.go
│   ├── sampler/{host.go,disk.go,proc.go}
│   ├── logtail/{tail.go,rotate.go}
│   ├── logparse/{nginx.go,routes.go,errorlog.go}
│   ├── aggregate/tdigest.go
│   ├── nginxmgr/{include.go,apply.go}
│   ├── actions/{registry.go,restart.go,ratelimit.go,ipblock.go}
│   └── ttl/store.go
├── ml/
│   ├── pyproject.toml
│   ├── opsis_ml/{__init__.py,server.py,features.py,anomaly.py,forecast.py,logtemplates.py,registry.py,train.py}
│   ├── scripts/evaluate.py
│   ├── tests/test_*.py
│   └── model/README.md
├── web/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── tailwind.config.ts
│   ├── index.html
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── styles.css
│       ├── api/{client.ts,types.ts,sse.ts}
│       ├── hooks/useLive.ts
│       ├── pages/{SetupWizard,Projects,ProjectOverview,SiteDetail,ServerDetail,Incidents,IncidentDetail,Fixes,AddServer,Alerts,System,WallBoard}.tsx
│       └── components/{StatusTile,RouteTable,LatencyChart,PhaseBar,Sparkline,CoveragePanel,EvidenceList,ApproveDialog}.tsx
├── firmware/opsis_mcu/opsis_mcu.ino
├── deploy/
│   ├── appliance/{install-on-board.sh,systemd/,kiosk/,avahi/,firstboot/,update/}
│   ├── agent/{install.sh,uninstall.sh,agent.example.yaml,systemd/,sudoers.opsis,nginx/}
│   └── dev/{docker-compose.yml,demo-app/,agent.yaml}
├── scripts/{gen-proto.sh,build-arm64.sh,deploy-board.sh,fault-inject.sh,loadtest.js,release.sh}
├── testdata/{access-logs/,memory/,probes/}
├── tests/{integration/,e2e/,failure/}
└── docs/
    ├── architecture.md
    ├── threat-model.md
    ├── support-matrix.md
    ├── runbook.md
    ├── hardware-notes.md
    └── adr/
Generated files, dependency lockfiles, trained model artifacts, runtime databases, and release binaries should be handled deliberately. Do not commit private secrets, local credentials, .env files, the active SQLite database, Python virtual environments, or generated model binaries unless there's an explicit reason.
4. File-by-file responsibilities and proposed functions
The function names below are suggestions to make the architecture concrete. Keep exported APIs small and agree on request/response types before multiple people implement connected components. Private helper functions can be added as required by the implementation.
4.1 Root tooling, contracts, and build files
File	Responsibility	Expected behavior / commands
README.md	Project overview, supported modes, architecture, safe-use rules, build order, and run commands.	Keep implementation status honest; update commands as they become real.
Makefile	Single entry point for common developer tasks.	Planned targets: proto, dev, test, lint, build-board, build-agent, deploy, fault. Each target invokes scripts/tools and returns a failing exit code on errors.
go.mod, go.sum	One Go module and pinned dependency checksums.	Shared by all Go executables and packages. Run go mod tidy after imports/dependencies are in place.
buf.yaml	Protobuf module, lint, and breaking-change policy.	Defines module layout and compatibility rules.
buf.gen.yaml	Go and Python code generation plugins and output directories.	Generated RPC code goes to gen/; never hand-edit it.
.golangci.yml	Go lint rules.	Configure reasonable correctness, error handling, and style checks; avoid enabling rules that the team cannot maintain.
.editorconfig	Consistent whitespace, newline, and encoding rules.	Shared conventions across editors.
opsis.code-workspace	Multi-root VS Code workspace.	Include repository root, web/, and ml/ folders.
.vscode/settings.json	Editor formatters and project defaults.	Configure Go formatting, Python Ruff, Prettier/TypeScript, and test discovery.
.vscode/launch.json	Debug profiles.	Launch opsisd, local opsis-agent, opsis-ml, and the web dev server when each becomes runnable.
.vscode/tasks.json	Editor tasks.	Wrap Make targets; avoid duplicating build logic in task definitions.
.vscode/extensions.json	Suggested extensions.	Go, Python/Pylance, Ruff, ESLint, Prettier, Tailwind, Buf, REST Client, SQLite viewer, Docker, YAML, Playwright, Remote-WSL and Remote-SSH.
.github/workflows/ci.yml	Tests, lint, and compile checks on pull requests/pushes.	Run Go tests/race tests, Python tests/lint, frontend checks, and protobuf lint where configured.
.github/workflows/release.yml	Build and publish release artifacts.	Cross-compile intended targets, checksum/sign artifacts, and publish only after validation.
proto/agentlink/v1/agentlink.proto	Board-agent protocol.	Define pairing and a bidirectional stream for telemetry, heartbeats, route summaries, events, acknowledgements, and typed expiring actions.
proto/ml/v1/ml.proto	Go-Python ML protocol.	Define Score, Forecast, LogTemplates, Train, and Health RPCs with versioned features and bounded inputs.
gen/go/, gen/python/	Generated RPC stubs.	Produced by Buf/protoc workflow; never manually edit.


4.2 Executables in cmd/
cmd/opsisd/main.go — board daemon / composition root
Responsible for composition, not business logic. Proposed sequence:
1. Load and validate configuration.
2. Initialize structured logging and redaction.
3. Open SQLite and run numbered migrations.
4. Construct repositories and core services.
5. Construct probes, ingestion, aggregation, detection, agent-link, ML client, notifications, and policy services.
6. Start background workers and the HTTP/SSE server.
7. Handle SIGINT/SIGTERM, stop accepting requests, cancel workers, flush writes, and close resources gracefully.
Proposed functions: main(), run(ctx context.Context) error, buildApp(ctx context.Context, cfg config.Config) (*App, error), and shutdown(ctx context.Context) error (exact type placement may vary). Keep dependency construction here so packages don't import each other in circles.
cmd/opsis-agent/main.go — user's server agent
Loads agent.yaml, initializes pairing/certificates, host sampler, log tailing/parsers, route aggregation, TTL manager, typed action registry, and the mTLS gRPC server. Proposed functions: main(), run(ctx context.Context) error, and a shutdown routine that stops samplers/tailers and closes the gRPC server.
cmd/opsisctl/main.go — local operations CLI
Proposed command families: status, doctor, support-bundle, config-check, pair, update, rollback, backup-export, and factory-reset. Use a CLI framework only if useful; never allow arbitrary shell text from a remote request to become an OS command. Destructive commands should require deliberate local confirmation and should have clear dry-run/backup behavior where applicable.
4.3 Shared board-side packages in internal/
Configuration, logging, and core models
File	Purpose	Proposed functions/types
internal/config/config.go	Environment/file configuration, defaults, validation, file paths, ports, intervals, concurrency, retention settings, and feature switches.	Load() (Config, error), Validate(Config) error; fail early on invalid or unsafe configuration. Never log secrets.
internal/logx/log.go	Structured logging, consistent fields, and secret redaction.	New(cfg Config) (*slog.Logger, error), Redact(string) string, log-field sanitizers. Rotation can be delegated to the service manager/logging setup rather than overbuilt.
internal/core/models.go	Shared domain entities and value types.	Types such as Project, Site, Route, ProbeResult, MetricSample, Rollup, Agent, Incident, Evidence, CauseHypothesis, Policy, ActionRun, User, Setting, and ModelInfo. Define units and timestamps explicitly.
internal/core/errors.go	Stable domain-level errors.	Sentinel/wrapped errors such as not found, conflict, validation, forbidden, unavailable, and stale evidence; map them to HTTP status codes only at the API boundary.


API, sessions, and live updates
File	Purpose	Proposed functions
internal/api/server.go	HTTP server lifecycle, timeouts, health/readiness endpoints, embedded static web files in production.	NewServer(...), Handler(), ListenAndServe(ctx), Shutdown(ctx), HealthHandler, ReadyHandler.
internal/api/routes.go	Central route registration and versioned route layout.	RegisterRoutes(mux, deps); bind each endpoint to a handler and middleware.
internal/api/middleware.go	Request ID, panic recovery, body-size limits, request logging, security headers, auth checks, CSRF, and rate limits where appropriate.	Middleware constructors such as RequestID, Recover, RequireAuth, RequireCSRF, MaxBodyBytes.
internal/api/auth.go	Local admin first-run setup, password verification, sessions, logout, and session expiry.	SetupAdmin, Login, Logout, RequireSession; use Argon2id password hashing and secure cookies.
internal/api/sse.go	Server-Sent Events for dashboard refresh.	StreamHandler, event publisher/subscriber, heartbeat, reconnect-safe event IDs where needed. Do not hold unbounded client queues.
internal/api/dto.go	Stable JSON request/response shapes and validation.	DTO types, conversion/validation functions; don't expose raw database rows accidentally.
internal/api/handlers_projects.go	Create/list/read/update projects.	ListProjects, CreateProject, GetProject, UpdateProject.
internal/api/handlers_sites.go	Site and route management, coverage, metrics queries.	ListSites, CreateSite, GetSite, UpdateSite, AddRoute, UpdateRoute, GetCoverage, GetSiteMetrics.
internal/api/handlers_agents.go	Agent pairing/status/revocation endpoints.	CreatePairing, GetAgent, RevokeAgent. Do not reveal reusable pairing secrets after initial display.
internal/api/handlers_incidents.go	Incident list/detail, status and diagnosis.	ListIncidents, GetIncident, GetIncidentTimeline, GetDiagnosis, DismissIncident as required by the contract.
internal/api/handlers_policies.go	Per-project policy and automation mode configuration.	ListPolicies, CreatePolicy, UpdatePolicy, DisablePolicy; validate action types and limits.
internal/api/handlers_actions.go	Action history, approve/reject and outcomes.	ListActions, ApproveAction, RejectAction, GetAction. Approval must bind to a specific proposed action and its parameters.
internal/api/handlers_alerts.go	Alert channel settings and test notifications.	GetAlertChannels, UpdateAlertChannels, TestAlertChannel; never return secret tokens in cleartext.
internal/api/handlers_system.go	System status, Wi-Fi/time setup, update, backup, reset.	GetSystemInfo, UpdateNetwork, UpdateTime, CreateBackup, StartUpdate, RequestFactoryReset, with strict authorization.


Handlers should orchestrate the request and call service/repository methods. No raw SQL, long-running probes, ML execution, or remediation implementation belongs in HTTP handlers.
SQLite storage and retention
File	Purpose	Proposed behavior/functions
internal/store/db.go	Open SQLite and set WAL, busy timeout, connection limits, and safe pragmas.	Open(ctx, path) (*sql.DB, error), Ping(ctx), Close(). The chosen SQLite driver and build mode must match ARM64 deployment needs.
internal/store/migrate.go	Apply numbered migrations in order and record schema version.	Migrate(ctx, db) error, CurrentVersion(ctx). Fail startup clearly if a migration cannot be applied.
internal/store/projects_repo.go	Parameterized project reads/writes.	Create, GetByID, List, Update, Delete only if deletion policy is defined.
internal/store/sites_repo.go	Site configuration reads/writes.	Create, GetByID, ListByProject, Update, SetEnabled.
internal/store/routes_repo.go	Route definitions.	Create, ListBySite, Update, Delete; validate methods, paths, expected status and optional keyword.
internal/store/probes_repo.go	Batch persistence and bounded querying of raw probe results.	InsertBatch, QueryByRouteRange, DeleteExpired. Keep query windows bounded.
internal/store/rollups_repo.go	Rollup storage.	UpsertBucket, QueryBuckets, DeleteExpired; writes should be batched.
internal/store/agents_repo.go	Paired agent metadata and status.	CreatePairedAgent, Get, List, UpdateLastSeen, Revoke. Never store private keys as ordinary metadata.
internal/store/incidents_repo.go	Incident lifecycle, timeline, and evidence.	Open, Merge, Get, List, AddEvidence, Resolve, UpdateSeverity.
internal/store/policies_repo.go	Policies and action limits.	Create, Get, ListByProject, Update, Disable.
internal/store/actions_repo.go	Immutable-ish action history and state transitions.	CreateRun, RecordApproval, RecordResult, RecordVerification, RecordRevert, List. Preserve auditability.
internal/store/audit_repo.go	Append-only audit events.	Append, ListBounded; avoid update/delete APIs except a documented retention/export policy if required by law or operations.
internal/store/users_repo.go	Admin and session-related persistence.	CreateAdmin, GetAdminByName, session create/lookup/revoke as appropriate; store password hashes, not passwords.
internal/store/mlmodels_repo.go	Model versions, thresholds, and active/shadow state metadata.	RegisterModel, GetActiveModel, SetShadowState, ListVersions.
internal/store/outbox_repo.go	Durable optional sync queue.	Enqueue, ClaimBatch, MarkSynced, ReleaseForRetry; use idempotent event IDs.
internal/store/retention.go	Disk-budget and retention cleanup.	RunCleanup, DeleteExpiredRaw, CompactRollups, PreserveIncidentEvidence, CheckDiskBudget.
internal/store/migrations/001_core.sql	Projects, sites, routes, probe results, rollups, settings and users.	Idempotence/versioning is handled by the migration runner; add appropriate foreign keys and indexes.
internal/store/migrations/002_agents_metrics.sql	Agents and agent metrics/log-template metadata.	Includes paired agent metadata, summarized metrics, and log-template counts.
internal/store/migrations/003_incidents.sql	Incidents and evidence.	Store fingerprints, severity/state, start/resolved timestamps, and evidence references.
internal/store/migrations/004_policies_actions_audit.sql	Policies, action runs, and audit.	Store typed action parameters, policy mode, approvals, result, verification and expiry metadata.
internal/store/migrations/005_ml_models_outbox.sql	ML model metadata and sync outbox.	Ensure optional cloud sync is not required for local writes or detection.


Use prepared/parameterized SQL only. Establish foreign keys, indexes for common time-series queries, transactions for state transitions, and explicit maximum result windows. Do not load unbounded telemetry history into memory.
Route probing and SSRF defenses
File	Purpose	Proposed functions/behavior
internal/probe/scheduler.go	Per-site/per-route scheduling, jitter, bounded worker pool, cancellation, and retry timing.	Start(ctx), ScheduleRoute, RunDue, Stop; prevent overlapping requests per target if they would distort latency or exhaust resources.
internal/probe/http.go	HTTP/HTTPS request and latency-phase measurement.	Probe(ctx, Route) (ProbeResult, error) using net/http and httptrace; collect DNS, connect, TLS, TTFB, total duration, status, response bytes, keyword result, redirects.
internal/probe/tls.go	Certificate expiry and TLS handshake details.	CheckCertificate(ctx, host), DaysUntilExpiry; schedule certificate checks around hourly rather than per route tick.
internal/probe/dns.go	DNS resolution, timing, and answer changes.	Resolve(ctx, host), normalized result/error types.
internal/probe/selfcheck.go	Distinguish the appliance's own network failure from target outage.	CheckInternet(ctx), BoardConnectivityHealthy; suppress target-down incidents when external control probes show the board itself is offline.
internal/probe/fingerprint.go	Best-effort hosting/CDN clues for onboarding.	Fingerprint(ctx, url), returning evidence and confidence, never an authoritative or security-sensitive verdict.
internal/probe/ssrf.go	Outbound destination validation.	ValidateURL, ResolveAndValidate, ValidateRedirect, IsForbiddenIP; reject loopback, private, link-local, and cloud-metadata ranges by default and re-check every redirect/resolution.


Probe-engine requirements:
- Allow only http and https for ordinary URL checks.
- Revalidate redirect hops (with a small cap, e.g. five) and defend against DNS rebinding.
- Bound connection, TLS, total request time, response-body bytes, redirect count, retries, and concurrency.
- Avoid collecting response bodies except for the bounded optional keyword/field check.
- One short retry can reduce transient false alarms; do not create aggressive retry storms.
- Use consecutive-failure/hysteresis rules before declaring a route down.
- Never treat a missing sample as a successful zero-latency sample.
Agent connection, telemetry ingestion and aggregation
File	Purpose	Proposed functions/behavior
internal/agentlink/client.go	Board-side gRPC client.	Connect, Pair, Close; verify pinned certificate and connection state.
internal/agentlink/stream.go	Long-lived board-initiated stream and reconnect logic.	RunStream, HandleMessage, ResumeFromSequence; bounded backoff, heartbeats, sequence numbers, and idempotent message handling.
internal/agentlink/pairing.go	One-time token workflow.	CreatePairingToken, CompletePairing, RevokePairing; tokens expire and can be used once.
internal/agentlink/certs.go	Certificate creation/storage/validation.	GenerateClientCertificate, PinFingerprint, ValidatePeer, RevokeCertificate; protect private key material.
internal/ingest/pipeline.go	Validate, normalize, redact, and persist incoming telemetry.	IngestMetricBatch, IngestRouteStats, IngestLogSummary; accept only known schema versions and bounded batches.
internal/ingest/normalise.go	Normalize metric names, units, timestamps, and labels.	NormalizeMetric, NormalizeLabels; cap label cardinality.
internal/ingest/redact.go	Remove secrets/PII from data before persistence or display.	RedactLogLine, RedactFields; never persist request bodies, cookies, or Authorization headers.
internal/aggregate/window.go	Convert samples into time windows, preserve missing-data flags.	BuildWindow, CloseWindow; include sample count, completeness, and window timestamps.
internal/aggregate/rollup.go	Compute/persist minute-level summaries.	AggregateMinute, FlushRollups; mean/min/max/percentile and count calculations must be tested.
internal/aggregate/features.go	Build the versioned ML feature vector.	BuildFeatureVector, ValidateFeatureVector; feature names/order/version must match Python.


Detection and ML client
File	Purpose	Proposed functions/behavior
internal/detect/rules.go	Rule definitions and default thresholds for route-down, slow response, TLS expiry, error bursts, CPU saturation, restarts, traffic spikes, memory/disk risks.	EvaluateRules(window, baseline) []Signal; allow bounded per-site overrides.
internal/detect/baseline.go	Always-available baseline implementation.	UpdateBaseline, ExpectedRange, Deviation; initially rolling median and MAD, later hour-of-week EWMA where enough history exists.
internal/detect/forecast.go	Non-ML time-to-exhaustion fallback.	ForecastExhaustion(series, limit) with confidence/quality and minimum-data checks.
internal/detect/hysteresis.go	Avoid alert flapping.	ShouldOpen(kOfN), ShouldClose(mCleanWindows); handle gaps explicitly.
internal/detect/fusion.go	Combine rule signals and optional ML scores.	FuseSignals; rules remain authoritative for high-severity triggers; ML-only anomaly incidents are low severity and cannot trigger actions alone.
internal/detect/coverage.go	Resolve collectable metrics and explain unavailable ones.	ResolveCoverage(site, agent, integrations), returning enabled/unavailable metrics plus reasons and setup guidance.
internal/mlclient/client.go	Go client for the local Python gRPC service.	Health, Score, Forecast, LogTemplates, Train; use deadlines, payload limits, version checks, bounded concurrency and graceful fallback.


Rule examples from the design (tune only after test evidence): route down after two consecutive failed probes while board self-check passes; slow route if p95 is more than 3× baseline and at least baseline + 300 ms in 3 of 5 windows; TLS warning when expiry is under 14 days; use minimum sample counts for 5xx-rate rules; forecast memory/disk risk only with enough valid history.
Root cause, incidents, policy and actions
File	Purpose	Proposed functions/behavior
internal/rca/layers.go	Model the service path (DNS/TLS, CDN/proxy, web server, app, dependency, host, traffic).	BuildLayerChain(site); keep layer mapping explicit/configurable.
internal/rca/evidence.go	Gather fresh supporting, contradicting and missing evidence.	CollectEvidence(incident), ScoreEvidence; time-align observations and label source/time range.
internal/rca/rank.go	Rank cause hypotheses.	RankCauses(evidence) []Hypothesis; confidence must represent limited evidence honestly.
internal/rca/explain.go	Generate understandable explanation from structured evidence.	Explain(hypothesis); templates/rules preferred over ungrounded generated statements.
internal/incident/engine.go	Incident state and lifecycle orchestration.	ProcessSignals, OpenOrMerge, Escalate, Resolve; deduplicate and combine correlated events.
internal/incident/state.go	State-transition validation.	CanTransition, ApplyTransition; transitions should be explicit and testable.
internal/incident/dedupe.go	Fingerprints/cooldowns.	Fingerprint(signal), FindRelated; fingerprint includes site/agent, route/metric and rule. Merge related anomalies in a bounded time window (the proposed default is five minutes).
internal/policy/model.go	Typed policy model and action parameters.	Policy, ActionSpec, AutomationMode; never use arbitrary shell command strings.
internal/policy/evaluate.go	Decision function for notify/recommend/approval/execute.	Evaluate(ctx, incident, proposedAction) Decision; consider mode, action allowlist, enabled flag and policy limits.
internal/policy/guards.go	Hard safeguards.	CheckFreshEvidence, CheckPreconditions, CheckCooldown, CheckHourlyBudget, RequireApproval, CheckIdempotency.
internal/action/executor.go	Dispatch a typed action through the paired agent.	Execute(ctx, approvedAction) with idempotency key, expiry, policy ID and audit event.
internal/action/verify.go	Independently measure post-action state.	VerifyRecovery(ctx, actionRun); use probes/metrics rather than trusting only the agent's success response.
internal/action/revert.go	Roll back unsafe/ineffective changes.	Revert(ctx, actionRun, reason); record result and escalate if revert fails.


Notifications, appliance controls, optional sync and MCU
File	Purpose	Proposed functions/behavior
internal/notify/router.go	Select notification routes, quiet hours and deduplication.	NotifyIncident, TestChannel, per-channel result recording.
internal/notify/telegram.go	Telegram notifications.	SendTelegram; handle tokens as secrets and apply request timeouts.
internal/notify/email.go	SMTP notifications.	SendEmail; TLS and credential handling, bounded retries.
internal/notify/webhook.go	User-configured webhooks.	SendWebhook; protect against SSRF, constrain destinations, redact payloads and use timeouts.
internal/sysctl/wifi.go	First-boot/network setup through supported Debian network tooling.	ListNetworks, ApplyNetworkConfig; admin-only operation and safe failure behavior.
internal/sysctl/timesync.go	Time zone and time status.	GetTimeStatus, SetTimezone; don't let arbitrary input reach a shell.
internal/sysctl/update.go	Signed update and rollback orchestration.	CheckUpdate, VerifyRelease, ApplyUpdate, Rollback; atomic switch and health-gated commit.
internal/sysctl/backup.go	Export settings/configuration.	CreateBackup, RestoreBackup; validate archive paths and protect secrets.
internal/neonsync/outbox.go	Enqueue durable local events for eventual sync.	Enqueue, ClaimBatch, MarkSynced; write local state first.
internal/neonsync/worker.go	Retry/backoff and idempotent upsert sync.	RunSyncWorker, SyncBatch; never blocks the local detection path.
internal/neonsync/restore.go	Restore syncable configuration/history onto a replacement board.	FetchRemoteState, RestoreConfig; validate versions and don't restore unsafe action policies as active without review.
internal/mcu/bridge.go	Optional Linux-to-MCU serial bridge.	SetLEDState, Buzz, ReadButtonEvents, Heartbeat; application must operate if the MCU is absent.


4.4 Server-side agent (agent/)
The agent is installed on the user's Linux server, not on the board. It should run under a dedicated non-root account. Any operation requiring privilege must be narrowly delegated (e.g. exact allow-listed systemd unit or nginx test/reload action), and the agent must never expose a remote shell.
File	Responsibility	Expected functions/behavior
agent/config/config.go	Parse and validate agent.yaml.	Load(path), Validate(Config); check paths, listen address, certificate locations, allowlists and state directory.
agent/pairing/server.go	One-time pairing handshake.	StartPairing, ConsumeToken, RevokeBoard; reject expired/reused tokens.
agent/pairing/certs.go	Agent certificate and peer pin management.	LoadOrCreateIdentity, ValidateBoardCert, RevokeFingerprint.
agent/grpcserver/server.go	mTLS gRPC endpoint (design default port 7443).	Start, Stop, AuthenticatePeer, ServeStream; enforce certificate, connection and message limits.
agent/sampler/host.go	CPU, load, memory, swap, file descriptor and related host metrics.	SampleHost(ctx), emit unit-labelled bounded metrics.
agent/sampler/disk.go	Disk capacity/inodes.	SampleDisk(ctx); report filesystem identity and usage; avoid sensitive path inventory.
agent/sampler/proc.go	Allow-listed process/service RSS and restart observations.	SampleProcesses(ctx, configuredUnits); avoid unrestricted process command-line capture.
agent/logtail/tail.go	Follow configured log files without blocking web service.	RunTail(ctx, path), bounded line buffer, backpressure policy.
agent/logtail/rotate.go	Handle log rotation/reopen safely.	ReopenIfRotated, file identity tracking.
agent/logparse/nginx.go	Parse the configured OPSIS nginx access-log format.	ParseAccessLine; handle malformed lines without crashing.
agent/logparse/routes.go	Normalize concrete paths to configured route patterns.	NormalizeRoute(path, patterns); unknown paths go to a bounded other bucket.
agent/logparse/errorlog.go	Extract useful error templates from nginx error logs/journald.	ParseErrorLine, redact secrets, classify known patterns.
agent/aggregate/tdigest.go	Streaming latency percentiles.	Add, Quantile, Snapshot; validate p50/p95/p99 against reference datasets.
agent/nginxmgr/include.go	Render only the OPSIS-owned nginx include/zone.	RenderConfig, BackupCurrent; do not rewrite unrelated user configuration.
agent/nginxmgr/apply.go	Validate, apply, and revert nginx change.	TestConfig, Reload, Revert; run nginx validation before reload and restore previous configuration on failure.
agent/actions/registry.go	Registry of allowed typed actions.	LookupAction, ValidateParams; unknown action types are rejected.
agent/actions/restart.go	Restart a configured service.	RestartAllowlistedUnit; narrow sudoers rule, single attempt, cooldown, expiry and health check.
agent/actions/ratelimit.go	Temporary nginx rate limit.	ApplyRateLimit, RemoveRateLimit; conservative values, exemptions, nginx test, reload and server-side TTL.
agent/actions/ipblock.go	Temporary deny rule where specifically enabled.	BlockIP, UnblockIP; reject private/allow-listed addresses and apply a short expiry.
agent/ttl/store.go	Persist action expiry state across board outage and agent reboot.	SaveExpiry, RunExpiryWorker, ExpireDueActions; changes must expire even if the board is offline.


The agent sends summaries, not raw log files, to the board: roughly ten-second route windows with count, 4xx/5xx counts, bytes and p50/p95/p99 plus host samples/log-template counts. Sequence IDs and idempotency keys make reconnection safe.
4.5 Python ML service (ml/)
The ML process is an optional specialist. It must be memory-bounded (target described in the design: about 600 MB or less), isolated from action execution, and safe to restart independently.
File	Responsibility	Proposed functions/behavior
ml/pyproject.toml	Python package metadata, pinned runtime/dev dependencies and tool config.	Define Python version and dependencies such as grpcio, numpy, scikit-learn, scipy, drain3, joblib, Ruff and pytest. Verify ARM64 wheels and resource use on target.
ml/opsis_ml/__init__.py	Python package marker/version.	Export package version/schema constants if needed.
ml/opsis_ml/server.py	gRPC server on a Unix socket, with no public network listener.	Implement Health, Score, Forecast, LogTemplates, Train; validate sizes/schema before dispatch.
ml/opsis_ml/features.py	Versioned feature schema and missing-value policy.	validate_features, normalize_features, feature_names_for_version; reject wrong order/version and preserve missing-data semantics.
ml/opsis_ml/anomaly.py	Isolation Forest inference and per-feature deviation explanations.	score_window, derive_threshold, rank_feature_deviations; threshold from held-out/training quantiles, not arbitrary universal constants.
ml/opsis_ml/forecast.py	Time-to-exhaustion estimate using robust Theil–Sen regression on a sliding minimum.	forecast_exhaustion(series, limit); require sufficient samples, return uncertainty/quality, avoid GC sawtooth false positives.
ml/opsis_ml/logtemplates.py	Drain-style log template mining and novelty/rate metrics.	parse_line, update_templates, summarize_novelty; only process redacted/approved logs and bound state.
ml/opsis_ml/registry.py	Versioned joblib model metadata and model swapping.	load_model, register_model, activate_model, rollback_model; validate feature schema/model metadata and keep prior version available.
ml/opsis_ml/train.py	Train or retrain using validated historical normal windows.	load_training_windows, train_model, save_version; no model becomes active without evaluation/metadata.
ml/scripts/evaluate.py	Replay fault fixtures and evaluate the detector.	Report detection delay, false alarms per idle day, precision/recall, inference latency and memory.
ml/tests/test_features.py	Schema, unit, missing feature and invalid payload tests.	Assert malformed inputs fail safely.
ml/tests/test_anomaly.py	Score/threshold behavior tests.	Cover normal windows, injected anomalies, model not loaded and deterministic seeds where supported.
ml/tests/test_forecast.py	Forecast quality tests.	Test rising resource series, flat series, noisy/GC-like sawtooth, missing samples, and insufficient history.
ml/tests/test_server.py	RPC behavior tests.	Test payload bounds, schema rejection, deadlines, unavailable models, and health reporting.
ml/model/README.md	Model artifact policy and metadata.	Document artifact location/version and training process; don't commit model binaries by default.


ML development must begin in shadow mode: scores are logged and compared against deterministic rules while they are not allowed to change alert severity or execute actions. Use injected faults and observed false-alarm metrics before considering wider use. If opsis-ml fails, opsisd continues rule/baseline detection.
4.6 Browser console (web/)
The browser is a client of opsisd; it must not connect directly to SQLite, Neon, the agent, or the Python service. The production web build is embedded into the Go daemon after the embed integration is implemented.
File	Responsibility	Expected functions/behavior
web/package.json	Frontend dependencies and scripts.	dev, build, typecheck, lint, and test scripts once configured.
web/vite.config.ts	Vite dev server and production build configuration.	Local API proxy can target Go; production output goes to the configured embed directory.
web/tsconfig.json	TypeScript compiler settings.	Strict types, modern target, no unchecked assumptions where practical.
web/tailwind.config.ts	Tailwind content/theme configuration.	Include all UI source paths.
web/index.html	SPA HTML entry.	Defines app root and metadata; no secrets.
web/src/main.tsx	React bootstrap.	Initialize React, router, query cache/providers and root error boundary as chosen.
web/src/App.tsx	Route layout, app navigation, stale-data/connectivity state.	Register screen routes and provide shared layout.
web/src/styles.css	Global styles and theme tokens.	Accessible contrast, responsive layout, kiosk/wall-board sizing.
web/src/api/client.ts	Typed REST client.	Functions such as getHealth, listProjects, createProject, listSites, getMetrics, listIncidents, approveAction; use bounded timeouts and consistent API errors.
web/src/api/types.ts	Frontend types matching versioned API DTOs.	Project, Site, Route, ProbeResult, MetricPoint, Incident, Evidence, CauseHypothesis, Policy, ActionRun, etc. Keep in sync with the backend contract.
web/src/api/sse.ts	SSE connection.	subscribeToEvents, reconnect/backoff, parse typed events; don't leak auth tokens into URL parameters.
web/src/hooks/useLive.ts	Merge live updates into query/cache state.	Hook should clean up subscriptions and handle stale/disconnected state.
web/src/pages/SetupWizard.tsx	First-run Wi-Fi/network, time zone, admin password and alert setup.	Clear step validation and recovery when a configuration action fails.
web/src/pages/Projects.tsx	Project list/create.	Empty/loading/error states and safe create form.
web/src/pages/ProjectOverview.tsx	Cross-site overview.	Status tiles, active incidents, agent health, and update freshness.
web/src/pages/SiteDetail.tsx	Route status, latency phase chart, TLS expiry, route edit and coverage.	Show why metrics are unavailable, not zero; display source/time range.
web/src/pages/ServerDetail.tsx	Server host charts, forecasts, per-route traffic/errors, log templates.	Make clear the data source is the paired agent and show last-seen time.
web/src/pages/Incidents.tsx	Incident list and filters.	Severity/state and bounded pagination.
web/src/pages/IncidentDetail.tsx	Timeline, evidence, ranked causes, recommendation, approval/reject/dismiss.	Display supporting/contradicting/missing evidence and exact action parameters before approval.
web/src/pages/Fixes.tsx	Automation modes, allowed action types, limits and action history.	Expose the safe defaults and per-site enablement; no hidden autonomous mode.
web/src/pages/AddServer.tsx	Install instructions, one-time pairing and pairing status.	Display one-time secret only when created; revoke/re-pair instructions.
web/src/pages/Alerts.tsx	Telegram/email/webhook config and quiet hours.	Secret values are write-only/masked; test notification feedback.
web/src/pages/System.tsx	Network, time, software update, backup, cloud sync and reset.	Confirm risky operations, show progress and rollback state.
web/src/pages/WallBoard.tsx	Large status tiles and auto-rotating overview.	Read-only wall display, good contrast, no mouse-dependent controls.
web/src/components/StatusTile.tsx	Reusable service/route state tile.	Explicit healthy/warning/critical/unknown states.
web/src/components/RouteTable.tsx	Route list and status/latency details.	Sorting/filtering and bounded rendering.
web/src/components/LatencyChart.tsx	Time-series and latency percentile chart.	Missing points stay gaps, not zeros.
web/src/components/PhaseBar.tsx	DNS/connect/TLS/TTFB/total timing visualization.	Use common units and handle unavailable phases.
web/src/components/Sparkline.tsx	Compact trend line.	Accessible label and empty-data state.
web/src/components/CoveragePanel.tsx	Metric availability and explanation.	For each metric, show available/unavailable and how to enable it.
web/src/components/EvidenceList.tsx	Evidence source, timestamp, time range and details.	Escape untrusted log content.
web/src/components/ApproveDialog.tsx	Deliberate confirmation for a proposed action.	Display target, parameters, reason, expiry, risks and expected verification before posting approval.


The UI must include loading, empty, stale, disconnected, partial-data and API-error states. It must not imply that an action succeeded until the backend returns an outcome.
4.7 Optional MCU firmware (firmware/)
firmware/opsis_mcu/opsis_mcu.ino implements the optional board-facing indicator behavior: green/amber/red LED state, buzzer on critical incident if configured, button acknowledgement, and a heartbeat watchdog. It should communicate through the supported Arduino/Linux bridge and fail inertly if unavailable. The MVP must operate without this MCU integration.
4.8 Deployment files (deploy/)
File/directory	Responsibility
deploy/appliance/install-on-board.sh	Install/check required Debian packages, copy verified binaries/web/ML/migrations, create dedicated users/directories, install service units, kiosk, and Avahi configuration. Show what will change and provide an uninstall/recovery path.
deploy/appliance/systemd/opsisd.service	Start the Go board service, define restart policy, user/permissions, startup dependencies and sandbox restrictions.
deploy/appliance/systemd/opsis-ml.service	Start the Python ML process separately with memory limits (design target about 600 MB or less), filesystem/process restrictions and restart policy.
deploy/appliance/kiosk/opsis-kiosk.desktop	Desktop auto-start entry for kiosk mode.
deploy/appliance/kiosk/kiosk.sh	Wait for backend readiness then open Chromium fullscreen at local dashboard; recover after browser exit and configure screen blanking.
deploy/appliance/avahi/opsis.service	Advertise opsis.local where mDNS is supported.
deploy/appliance/firstboot/firstboot.sh	Check the first-run flag and start the setup wizard without embedding default credentials.
deploy/appliance/firstboot/opsis-firstboot.service	Run first-boot setup initialization once.
deploy/appliance/update/opsis-update.sh	Verify signed/checksummed release, stage new version, switch atomically, run health checks, and roll back on failure.
deploy/agent/install.sh	Detect architecture, verify artifact checksum/signature, create dedicated opsis user/groups, install service and narrow sudoers policy, then initiate pairing.
deploy/agent/uninstall.sh	Stop/remove agent and its own files/units, explain any retained state/config, and avoid deleting unrelated nginx config.
deploy/agent/agent.example.yaml	Example configuration for gRPC listening address, pinned board fingerprint, nginx paths, allow-listed units and state directory. Contains no real secrets.
deploy/agent/systemd/opsis-agent.service	Run the server agent under its dedicated account with restart/resource/security restrictions.
deploy/agent/sudoers.opsis	Exact allowed privileged commands and configured units; never allow a general shell. Validate executable paths for each target server.
deploy/agent/nginx/opsis-zone.conf	Example nginx rate-limit zone owned by OPSIS.
deploy/agent/nginx/limit.conf.tmpl	Empty/default OPSIS include template for temporary rate-limit or deny rules.
deploy/agent/nginx/log_format.conf	Proposed nginx access-log format containing only needed fields and avoiding query tokens/request bodies.
deploy/dev/docker-compose.yml	Local demo/test dependencies such as nginx, demo app, load generator, and fault toggle. It is a development environment, not a requirement on the UNO Q appliance.
deploy/dev/demo-app/	Small disposable test app for healthy/slow/error/fault scenarios.
deploy/dev/agent.yaml	Non-secret local agent config for the demo stack.


Prefer native Debian/systemd services for the first appliance release. Docker Compose is for the reproducible local test stack unless board resource testing justifies a different decision.
4.9 Scripts, test fixtures, tests, and docs
File/directory	Responsibility
scripts/gen-proto.sh	Run Buf lint/breaking checks and generate Go/Python RPC code.
scripts/build-arm64.sh	Build board binaries for linux/arm64, build web assets, and package artifacts. Confirm CGO/dependency constraints before choosing static build flags.
scripts/deploy-board.sh	Deploy to opsis.local or a configured board over SSH/rsync, restart services and check health. Require an explicit target; do not assume hostnames/credentials.
scripts/fault-inject.sh	Run explicitly selected fault scenarios against the local disposable demo stack. Never point fault injection at a production host by default.
scripts/loadtest.js	k6 load-test scenarios for the demo app; set conservative defaults and require an explicit target override for other environments.
scripts/release.sh	Package binaries, web, Python environment/artifacts, checksums and signatures after tests pass.
testdata/access-logs/normal.log	Replay fixture for typical traffic.
testdata/access-logs/5xx-burst.log	Replay fixture for error spikes.
testdata/access-logs/flood.log	Replay fixture for high-rate/abusive-pattern testing; synthetic or sanitized only.
testdata/memory/memory-leak.csv	Synthetic resource-growth series to test forecasts.
testdata/probes/probe-replay.json	Deterministic probe observations for rule and dashboard tests.
tests/integration/	Cross-package tests for probing, persistence, pairing, stream reconnection and telemetry ingestion.
tests/e2e/	Browser-based flows such as setup, project/site creation, incident display and action approval.
tests/failure/	Fault/chaos tests for service restart, internet loss, Python ML outage, disk-full behavior and other recovery paths.
docs/architecture.md	Living architecture and data flow.
docs/threat-model.md	Threats, boundaries, assumptions, mitigations and open security issues.
docs/support-matrix.md	Tested OS/architecture/nginx and optional integration support matrix; distinguish tested from merely expected.
docs/runbook.md	On-call diagnosis, support bundle, backup, update, rollback, revoke-agent and recovery instructions.
docs/hardware-notes.md	Tested UNO Q image, hub/monitor compatibility, RAM/storage measurements, networking and boot quirks.
docs/adr/	Architecture Decision Records; one numbered file per important decision with context, decision and consequences.


5. End-to-end data flow
Primary Mode B telemetry cycle
1. Agent setup and pairing: the user registers a Linux/nginx server in the console. A one-time pairing flow establishes peer identities and pinned certificates before telemetry is accepted.
2. Host collection: agent/sampler/ samples CPU/load, memory/swap, disk/inodes, file descriptors and configured process RSS/restart observations about every 5 seconds.
3. Log collection and route aggregation: agent/logtail/ follows nginx access/error logs, survives rotation, bounds buffers, redacts sensitive data, normalizes concrete URLs to registered route patterns, and aggregates about 10-second windows. The window contains request count, 4xx/5xx counts, bytes, and p50/p95/p99 latency. Unknown routes are grouped into a bounded other bucket.
4. Secure transport: the server agent sends metric batches, route summaries, redacted log-template counts, events and heartbeats over the board-initiated mTLS gRPC stream. Sequence numbers support reconnect/resume; duplicates are handled idempotently.
5. Ingestion and persistence: internal/ingest/ validates schema versions, names, units, timestamps and sizes, redacts fields, marks stale/missing data, and batches valid records into SQLite through repositories. Raw agent measurements have short retention; rollups support longer history.
6. Server-first detection: Go rules and baselines look for error bursts, CPU saturation coupled with latency growth, resource exhaustion, restart loops, and traffic spikes. The engine checks data quality and agent/board connectivity before generating incidents.
7. Incidents and root-cause analysis: the incident engine fingerprints and merges related anomalies. RCA aligns host metrics, route statistics, log patterns and optional external-probe evidence by time; it returns ranked hypotheses with supporting, contradictory, missing and stale evidence.
8. Dashboard and alerts: the Go API serves bounded REST queries and sends live changes using SSE. Telegram/email/webhook notifications can link to the incident evidence and current agent health.
9. Later, authorized action: policy checks the operation mode, exact allowlist, fresh evidence, preconditions, approval, idempotency, cooldown and hourly budget. The agent independently validates and executes a typed, expiring command. OPSIS verifies recovery; failures lead to revert/escalation and an audit record.
Supporting external-probe cycle
internal/probe/ runs scheduled HTTP/DNS/TLS checks for registered routes. Phase timings (DNS, connect, TLS, time-to-first-byte and total time) provide independent context. A board internet self-check distinguishes an appliance/network outage from a target outage. Every initial URL and redirect is subject to SSRF and destination safety checks.
Optional application health and cloud sync
- An application health endpoint can report application/dependency timings when agent installation is not possible; it does not expose private server statistics unless the app explicitly provides them.
- Neon synchronization is last/optional. Eligible config, incidents, audit records and minute rollups enter a local outbox and sync idempotently in the background. Raw probes and detailed agent measurements remain local by default; cloud failure never stops collection or detection.
Latency interpretation
- Slow DNS points toward name resolution or the resolver path.
- Slow connect/TLS points toward network establishment or TLS issues.
- Slow time-to-first-byte with otherwise healthy network phases suggests the server/upstream took longer to respond.
- Fast first byte but slow total time may indicate a large response or transfer-path issue.
These are diagnostic clues, not definitive root causes. The UI should retain phase values and the explanation's evidence sources.
6. API and RPC contracts
All routes below are planned API contracts, not a claim that they are already implemented. Use versioned /api/v1 endpoints. List endpoints must be paginated and time-series queries must have explicit time/resolution bounds. Every mutating endpoint requires an authenticated session, including requests from the LAN.
REST API
Method	Endpoint	Purpose
GET	/api/v1/health	Liveness check for the Go process.
GET	/api/v1/ready	Readiness check, including SQLite and relevant service status. ML unavailable should be surfaced but should not make core monitoring nonfunctional.
POST	/api/v1/auth/setup	First-run admin setup; available only when not configured.
POST	/api/v1/auth/login	Authenticate local admin.
POST	/api/v1/auth/logout	Revoke current session.
GET, POST	/api/v1/projects	List/create projects.
GET, POST	/api/v1/projects/{id}/sites	List/add sites for a project.
GET, PUT	/api/v1/sites/{id}	Read/update site configuration.
GET, PUT	/api/v1/sites/{id}/routes	Read/update route definitions (creation/deletion semantics should be explicitly represented in implementation DTOs).
GET	/api/v1/sites/{id}/coverage	Enabled and unavailable metrics, including reasons and the setup required.
GET	/api/v1/sites/{id}/metrics?from=&to=&res=	Bounded time-series query. Validate time span and resolution.
POST	/api/v1/agents/pairing	Create short-lived one-time pairing token/instructions.
GET	/api/v1/agents/{id}	Read paired-agent status/last seen/version.
DELETE	/api/v1/agents/{id}	Revoke an agent pairing.
GET	/api/v1/incidents	List incidents with pagination and filters.
GET	/api/v1/incidents/{id}	Read incident, timeline, evidence and ranked causes.
GET, POST	/api/v1/projects/{id}/policies	Read/create automation policies.
PUT	/api/v1/projects/{id}/policies/{policyId}	Update policy with typed actions and guardrails.
POST	/api/v1/actions/{id}/approve	Approve an exact proposed action and parameters.
POST	/api/v1/actions/{id}/reject	Reject a proposed action.
GET	/api/v1/actions	Action history, expiry, verification and revert outcomes.
GET, PUT	/api/v1/alerts/channels	Read/update configured alert channels (secret fields must be masked/write-only).
POST	/api/v1/alerts/test	Send a test notification.
GET	/api/v1/system	Read device/service status.
POST	/api/v1/system/wifi	Apply a validated network configuration.
POST	/api/v1/system/update	Trigger verified update workflow.
POST	/api/v1/system/backup	Create configuration backup/export.
POST	/api/v1/system/reset	Request guarded factory reset.
GET	/api/v1/stream	SSE stream for route/agent state, incident updates and action progress.


Use a consistent JSON error shape, request IDs, validation errors that identify invalid fields without leaking secrets, and stable response DTOs. Define exact pagination and SSE event contracts before frontend implementation.
Agent gRPC contract (proto/agentlink/v1/agentlink.proto)
The design defines a board-initiated mTLS connection to the server agent. Planned service methods/messages include:
- Pair(PairRequest) returns (PairResponse) — exchange a short-lived pairing token and establish trusted identities.
- Stream(stream BoardMsg) returns (stream AgentMsg) — long-lived bidirectional telemetry/control stream.
- Board-to-agent message categories: Hello, SetRoutes, SetPolicy, RunAction, Ack.
- Agent-to-board message categories: MetricBatch, RouteStatsBatch, LogSummary, Event, ActionResult, Heartbeat.
- Route summary fields include route ID/pattern, window start/duration, request count, 4xx/5xx counts, p50/p95/p99, and bytes.
- Every actionable request contains an idempotency key, expiry timestamp, and authorizing policy ID. Stream sequence numbers support safe reconnect/resume.
Go-to-Python gRPC contract (proto/ml/v1/ml.proto)
Planned RPCs:
- Score(ScoreRequest) returns (ScoreReply) — versioned feature window to anomaly score and useful per-feature deviations.
- Forecast(ForecastRequest) returns (ForecastReply) — resource series and limit to time-to-exhaustion estimate and confidence/quality.
- LogTemplates(LogRequest) returns (LogReply) — redacted log lines or summaries to template/count/novelty information.
- Train(TrainRequest) returns (TrainReply) — controlled request for training/evaluation; production activation requires model validation.
- Health(Empty) returns (HealthReply) — service and model readiness.
The Python server binds to a local Unix socket rather than a public TCP listener. Payload sizes, deadlines, feature schema versions and error responses must be defined and tested. Use Buf/protoc generation into gen/ and never hand-edit generated files.
7. Local database design and retention
SQLite runs on the appliance and remains authoritative. The initial conceptual entities are:
Table	Purpose / notable fields
projects	id, name, created_at.
sites	id, project_id, name, base_url, interval_s, timeout_ms, enabled.
routes	id, site_id, method, path, optional route pattern, expect_status, expect_keyword, critical.
probe_results	route_id, timestamp, success/status, DNS/connect/TLS/TTFB/total milliseconds, bytes and bounded error.
rollups	scope/scope ID, metric, bucket, count, sum/min/max and percentile columns.
agents	IDs/project/name/host, certificate fingerprint, pairing time, last seen, version and status.
agent_metrics	Agent ID, timestamp, metric name/value, bounded labels JSON. Keep short retention.
log_templates	Agent ID/template ID, redacted template text and first/last seen/count.
incidents	Project, fingerprint, times, severity/status/title and ranked cause data.
incident_evidence	Incident ID, evidence kind/reference/time/detail JSON.
policies	Project/target, trigger, action type/parameters, mode, cooldown, per-hour limit and enabled flag.
action_runs	Incident/policy, start time, typed command, result, verification, revert time and details.
audit	Timestamp, actor, event, target and detail JSON; append-only in normal operation.
settings, users	Device settings and local admin password hash (Argon2id).
ml_models	Service key, version, training time, threshold, artifact path and shadow flag.
outbox	Durable event ID/table/row/op/payload/time/sync state for optional Neon sync.


Migration files in the canonical architecture live under internal/store/migrations/ and are versioned/applied by internal/store/migrate.go. Avoid maintaining a second independent root-level migration history.
Retention policy (design starting point)
- Raw probe results: about 48 hours.
- One-minute rollups: about 30 days.
- Detailed agent metrics: short retention; confirm the final interval based on board storage measurements.
- Keep evidence referenced by open incidents even if its normal raw-data retention window expires; copy/summarize evidence as needed.
- Check storage budget and expose disk pressure to the user before writes fail.
- Batch writes; use WAL; do not write one permanent DB row every second for each metric.
These are initial design targets and must be confirmed by workload and eMMC wear testing on the actual board.
8. Detection, diagnosis, and remediation rules
Baselines and windows
- During initial warm-up (design target: 30–60 minutes), show that ML/baselines are learning and avoid high-confidence claims from inadequate history.
- Begin with rolling median and Median Absolute Deviation (MAD), then consider hour-of-week EWMA mean/variance after adequate data accumulates (the design mentions around a week).
- Preserve data-quality/missing-window flags. Missing data is not equivalent to healthy or zero.
- Use hysteresis: require a number of bad windows to open and a number of clean windows to clear an incident.
- Record sample counts and baseline versions in evidence so the result can be reproduced.
Example rule triggers from the HLD/LLD
Rule	Example default trigger	Extra checks
Route down	Two consecutive failed probes.	Board self-check must pass; suppress site-down alerts if the appliance's own internet is unavailable.
Route slow	p95 > 3× baseline and > baseline + 300 ms in 3 of 5 windows.	Require usable sample count and complete windows.
Error burst	5xx ratio > 5% for about two minutes with at least 20 requests.	Available for app/server traffic where request counts are collected; URL probing alone cannot supply the full request denominator.
TLS expiry	Certificate expires in under 14 days.	Distinguish expired/invalid certificate from certificate-check failure.
Memory exhaustion	Forecast indicates under about 30 minutes to limit.	Requires server/host metrics and enough history.
Disk exhaustion	Forecast indicates under about 24 hours to limit.	Requires host access and robust slope/history.
CPU saturation	CPU > 90% for about five minutes while p95 is rising.	Requires host sampler.
Restart loop	At least three restarts in ten minutes.	Requires process/service observation.
Traffic spike	Request rate > 4× baseline with supporting traffic evidence.	Avoid asserting flood/abuse solely from latency.


Thresholds are starting values from the proposed architecture, not universal truths. Measure false alarms and detection delay on replay/injected faults before changing defaults or enabling automated responses.
Root-cause analysis
The RCA chain should consider, where data exists: DNS/TLS, CDN/proxy, web server, application, database/dependency, host resources, and traffic. Use onset order, magnitude, freshness and direct evidence. Examples:
- DNS/TLS failure with otherwise normal later phases: suggest DNS/certificate/CDN checks.
- nginx upstream connection refused and the service is down: application/service failure may be likely.
- Upstream timeout plus high CPU and rising p95 with flat request rate: application saturation is plausible.
- Database-related application error plus dependency latency/port evidence: database layer may rank higher.
- High memory/swap plus OOM journal event: host resource exhaustion may be strongly supported.
- Traffic burst from a small number of clients can support a traffic-abuse hypothesis, but rate-limiting recommendations need actual traffic evidence from the agent.
Return a ranked list of hypotheses with confidence, the supporting facts, contradictory facts, missing measurements and time range. Do not present an ML-generated sentence or a single score as proof.
Safe remediation state machine
DETECTED
   -> VALIDATE (persist incident; ensure board/self-check and evidence quality)
   -> POLICY_CHECK
        -> NOTIFY / RECOMMEND
        -> AWAIT_APPROVAL
        -> EXECUTE (typed allow-listed action, idempotency key, expiry)
   -> COOLDOWN
   -> VERIFY (independent probes/metrics)
        -> verified recovery -> RESOLVE
        -> verification failure -> REVERT -> ESCALATE
Minimum guards:
- Observe/Recommend are safe defaults; Auto is off until explicitly enabled for a particular action/target.
- No arbitrary shell command strings and no LLM-generated shell commands.
- Every action must be a typed, registered action with validated parameters.
- Evidence must be recent enough for the action and required preconditions must hold.
- Enforce explicit approval where configured, idempotency, cooldown and per-hour action budget.
- Restart only a user-allow-listed systemd unit. Prefer stateless services by default and verify service health after restart.
- Nginx rate limit: write only OPSIS-owned include files, test config (nginx -t), back up previous state, reload only after validation, and revert if test/reload fails or health worsens.
- Temporary rate limits and IP blocks have server-side expiry and survive agent/board restarts. A board outage must not leave a website throttled indefinitely.
- Exempt required health routes and configured trusted/owner IPs from rate-limiting rules; block private/allow-listed IPs from deny actions.
- Verify results independently; record actor, policy, evidence, parameters, result, verification and reversion in audit/action history.
Database actions and broad changes are alert/recommend-only unless a separately reviewed design and threat model authorizes otherwise.
9. Security and privacy
OPSIS monitors systems that may be production systems. The product must fail closed for privileged actions and prefer loss of optional capabilities over unsafe behavior.
Authentication and sessions
- First-run admin setup must require the user to set a password; no default shared password.
- Use Argon2id for password hashing.
- Use authenticated session cookies with HttpOnly, SameSite, secure attributes when HTTPS is enabled, expiration and logout revocation.
- Use CSRF tokens for every state-changing browser request.
- LAN access requires login. Don't assume the local network is trusted.
- Rate-limit login attempts and avoid detailed errors that reveal whether an account exists.
Target and SSRF safety
- Accept only expected URL schemes and validate hostnames, ports and destination addresses.
- Resolve and reject loopback, private, link-local and cloud-metadata destinations by default unless an explicit advanced configuration allows a known address.
- Revalidate every redirect and DNS resolution; prevent DNS-rebinding bypasses.
- Bound request deadlines, response bytes, redirects, retries, concurrency and payload processing.
- Do not send probes with user-provided credentials by default. If credentials are later added, treat them as secrets and scope them narrowly.
Agent and privileged actions
- Dedicated, non-root agent user; read-only log access via appropriate groups where possible.
- mTLS, paired certificate pinning, token expiry, revocation and rate-limited connections.
- Narrow sudoers entries for exact commands and allow-listed systemd unit names; never grant general shell/root.
- Action messages are structured protocol values with type, parameters, expiry, idempotency key and policy ID.
- The agent must independently enforce its local allowlist and expiry even when the board is disconnected.
Telemetry, logs and secrets
- Treat logs and remote responses as untrusted input; fuzz-test parsers and escape text in the UI.
- Never collect request bodies, cookies or Authorization headers.
- Redact access tokens, credentials and sensitive fields before storage, diagnosis, notification or ML input.
- Hash or truncate visitor IPs after a short window if client grouping is required; don't keep more identifying data than the feature requires.
- Encrypt integration secrets at rest with a device-specific key where implemented. Document the physical-access limitation: software encryption alone does not protect against an attacker with complete device/root access.
- Keep .env, pairing keys, private certificates, credentials, production logs and live database files out of Git.
Release and operations security
- Verify checksums/signatures on releases, use atomic updates, run health checks and roll back on failure.
- Installer scripts should explain changes, use pinned/verified downloads, validate paths, and provide uninstall guidance.
- Backup archives and restore paths must resist path traversal and unauthorized access.
- Set resource caps for background services and ensure storage-full behavior is tested.
- Record security assumptions and residual risks in docs/threat-model.md.
10. Technology stack
The following stack is the proposed final design. Versions and ARM64 package availability must be pinned/verified during implementation.
Layer	Planned choice	Notes
Appliance OS	Debian Linux on Arduino UNO Q 4 GB; XFCE desktop for kiosk	Verify the actual supported image/versions, hub, monitor, boot and RAM/storage behavior on hardware.
Board/server-agent/CLI	Go 1.22+, net/http, chi, slog, google.golang.org/grpc	Shared root Go module; build board for linux/arm64, agent for supported Linux architectures.
Local DB	SQLite with modernc.org/sqlite and WAL; migration library such as goose or golang-migrate	Pure-Go driver simplifies cross compilation; confirm performance and compatibility.
Optional cloud DB	Neon Postgres via pgx	Only for config/incidents/audit/rollups through outbox. Never required for detection.
RPC contracts	Protobuf + gRPC + Buf	mTLS for board-agent, Unix socket for board-ML. Generated sources are not hand-edited.
Probes	Go net/http, httptrace, crypto/tls, Go DNS library	Bounded time, concurrency, response size and redirect behavior.
Agent libraries	gopsutil, file tail library such as nxadm/tail or fsnotify, t-digest implementation such as caio/go-tdigest	Verify module maintenance, license, ARM64 and Linux compatibility before pinning.
ML	Python 3.11+, grpcio, NumPy, scikit-learn, SciPy, Drain3, joblib	Dedicated virtual environment/service, schema checks, memory limit and model evaluation.
Web	React, TypeScript, Vite, Tailwind, Recharts, TanStack Query, SSE	Production bundle embedded into opsisd after integration is implemented.
Kiosk/network	Chromium kiosk mode, systemd/desktop autostart, xset, Avahi/mDNS, NetworkManager	Browser opens local dashboard after backend readiness.
Notifications	Telegram Bot API, SMTP/email, webhooks	Secrets and timeouts are mandatory; webhook destinations need outbound-request safety.
MCU	Arduino sketch + supported Linux bridge	Optional; core service cannot depend on it.
Testing	Go tests/race detector, pytest, Playwright, Docker Compose demo, k6	Synthetic/replayable fault fixtures preferred for repeatability.
Release	GitHub Actions, Makefile, ARM64 cross-build, minisign or cosign	Publish checksums/signatures, test update/rollback.


Resource constraints
The design targets an UNO Q 4 GB board with limited CPU, memory and eMMC. Profile actual RAM, disk, temperature, CPU, ML inference latency, writes/hour, and frontend/browser use on the hardware. Do not assume a full Prometheus/Grafana/Loki deployment fits the appliance. Prefer summarized metrics, batching, rolling windows and bounded model sizes.
11. Development setup and commands
Recommended developer workflow
Develop on the laptop first; use the board for hardware integration and resource testing. On Windows, WSL2 or Git Bash may be used for compatible shell tasks, but Linux-specific systemd, nginx, and agent tests should run in a Linux environment (for example WSL2/Docker) and ultimately on a tested Linux server. Use Remote-SSH for on-board diagnostics.

Install the tools required for each part of the project:
- Go toolchain compatible with go.mod.
- Python 3.11+ and venv/pip; verify all selected dependencies on ARM64.
- Node.js LTS and npm for the web console.
- Buf/protoc and configured Go/Python generators for gRPC contracts.
- Git, Make, Docker/Compose for the local demo, and k6/Playwright when those tests are introduced.
- Arduino tooling only if implementing the optional MCU firmware.
Initial commands
From the repository root:
# Inspect the canonical layout and current changes
pwd
git status --short
find cmd internal agent ml web proto deploy scripts tests docs -maxdepth 3 -type d | sort

# Inspect the one root Go module (create only once)
cat go.mod

# Once Go source imports/dependencies are implemented
go mod tidy
go test ./...

# Once ML package configuration is implemented
python -m venv ml/.venv
# Activate the environment using the command for your shell, then:
python -m pip install -e 'ml/[dev]'
python -m pytest ml/tests

# Once the web package.json scripts have been filled in
cd web
npm install
npm run dev
npm run build
cd ..
pip install -e 'ml/[dev]' requires a matching optional dependency group to be configured in ml/pyproject.toml; adjust the command to the actual project's package configuration if it uses a different extras name. These commands are intended for the implementation stage and will not all work against empty placeholder files.
Planned Make targets
make proto                         # buf lint, breaking check, generate stubs
make dev                           # Go API, Vite dev server, ML service, local demo stack
make test                          # Go tests, race checks, pytest, frontend tests
make lint                          # Go lint, Ruff, ESLint/TypeScript
make build-board                   # frontend assets + linux/arm64 board binaries
make build-agent                   # agent binaries for linux/amd64 and linux/arm64
make deploy BOARD=opsis.local      # deploy and health-check on a configured board
make fault SCENARIO=flood          # run an explicit synthetic fault against demo stack
These targets must be implemented in the Makefile; do not assume they work until there are recipes and their tools are installed.
Board deployment notes
- The UNO Q target is Debian ARM64; local development can run on Windows/Linux but the release artifact must match the device architecture.
- Proposed installed paths: /opt/opsis/bin/opsisd, /opt/opsis/bin/opsisctl, /opt/opsis/web/, /opt/opsis/ml/, /etc/opsis/, and /var/lib/opsis/ for local state.
- Store configuration/secrets in root-owned or service-user-restricted files with restrictive permissions.
- Install opsisd and opsis-ml as separate systemd services. Start the kiosk after the Go API is ready.
- Test first boot, browser crash/restart, reboot, network loss, ML crash, disk near-full, update rollback, and actual display/hub compatibility on hardware.
- A non-CGO Go build flag is only valid if all selected dependencies (particularly the SQLite driver) support that build mode. Verify rather than copying a static-build command blindly.
12. Chronological implementation plan — Mode B first
Every phase must end with a runnable result and an explicit acceptance test. This sequence follows the revised HLD/LLD: the server agent and real host/nginx telemetry are built early; external probes support the main path; ML and live remediation remain later until inputs and safety controls are proven.
Phase	Focus	What to implement	Primary files	Exit criteria
0 — Development loop and hardware constraints	Establish toolchain and target budgets.	Pin development tools; test SSH and Linux/ARM64 builds; validate UNO Q image, hub, monitor and network; measure RAM, CPU, storage and Python footprint.	scripts/build-arm64.sh, scripts/deploy-board.sh, docs/hardware-notes.md	A sample ARM64 binary runs on the board; deployment path works; resource measurements are recorded.
1 — Control-plane foundation	Minimal board daemon.	Root Go module, config, structured logging, SQLite WAL/migrations, API, health/readiness, graceful shutdown and persistence tests. Keep feature scope small.	cmd/opsisd, internal/config, internal/logx, internal/store, internal/api	Daemon starts reliably; health/readiness endpoints respond; migrated data survives restart.
2 — Agent-local telemetry	Make the server agent useful independently.	Build cmd/opsis-agent; validated config; host/load/memory/swap/disk/inode/FD/process sampling; nginx access/error log tailing, rotation handling, route normalization and ~10-second percentiles.	cmd/opsis-agent, agent/config, agent/sampler, agent/logtail, agent/logparse, agent/aggregate	Agent runs on a Linux test server and emits correct, bounded local host and route metrics.
3 — Pairing and secure gRPC	Connect agent and appliance securely.	Define agentlink.proto; one-time pairing; certificate issuance/pinning/revocation; board-initiated mTLS stream; sequence numbers, heartbeats, reconnect/resume and idempotency.	proto/agentlink/v1/agentlink.proto, agent/pairing, agent/grpcserver, internal/agentlink, deploy/agent	Only paired identities connect; stream reconnects safely; revocation blocks new connections; no inbound board port is required.
4 — Ingest, persist and server dashboard	First flagship end-to-end vertical slice.	Validate metric schema/units, redact, batch-write to SQLite; build agent/server/route APIs and SSE; display CPU/memory/disk, request rates, 4xx/5xx and p95/p99.	internal/ingest, internal/aggregate, internal/store/*_repo.go, internal/api, web/src/pages/ServerDetail.tsx, web/src/components/*	Real nginx host and route metrics are visible end-to-end on the OPSIS dashboard. This is the first major product milestone.
5 — Supporting external probes	Independent cross-checks.	HTTP timing phases, DNS/TLS/status/keyword checks, SSRF controls, redirect revalidation and board-internet self-check. Compare agent and probe evidence.	internal/probe/*, site/route handlers	Probe evidence is safe and helps distinguish application/server failures from network/probe-path problems.
6 — Rules, incidents and alerts	Server-first operational value.	Rules for CPU/resource pressure, disk/memory exhaustion, 5xx bursts, restart loops and traffic spikes; deduplication and incident lifecycle; Telegram/email/webhooks.	internal/detect/rules.go, internal/incident/*, internal/notify/*, internal/api/sse.go	Inject a controlled 5xx/load fault and receive an incident with server/route evidence and an alert.
7 — Baselines and robust detection	Reduce false positives.	Rolling median/MAD, later hour-of-week EWMA, missing-data flags, hysteresis, Go forecast fallback, rule/ML fusion design and replay fixtures.	internal/detect/*, internal/aggregate/*, testdata/*	Fixtures produce expected incidents; false alarms per idle day are measured.
8 — Python ML in shadow mode	Evaluate ML without granting authority.	Versioned features, Isolation Forest, Theil-Sen resource forecasts, Drain log templates, local gRPC, deadlines/fallback and evaluation.	proto/ml/v1/ml.proto, ml/opsis_ml/*, internal/mlclient/client.go	Scores are logged but cannot trigger actions; detection delay/false alarms/precision/recall are compared; Go detection survives Python failure.
9 — Evidence-based root-cause analysis	Explain, do not overclaim.	Layer chain (edge/web/app/dependency/host/traffic), time alignment, supporting and contradictory evidence, confidence and plain-language explanations.	internal/rca/*, web/src/pages/IncidentDetail.tsx	A controlled nginx/upstream failure ranks a plausible layer with explicit evidence and uncertainty.
10 — Approval-gated remediation	Introduce narrow server changes only after evidence is trusted.	Typed allowlisted restart, temporary nginx rate limit and optional IP block; policy guards; approval UI; persistent TTL; audit; independent verification and revert.	internal/policy/*, internal/action/*, agent/actions/*, agent/ttl/*, agent/nginxmgr/apply.go	Unapproved/stale/duplicate actions are denied; temporary limits expire when the board is offline; failed nginx tests revert safely.
11 — Appliance readiness	Make it usable as a product.	First-boot wizard, network/time setup, Chromium kiosk, opsis.local, signed update/rollback, backup/export/reset and optional MCU integration.	deploy/appliance/*, internal/sysctl/*, web/src/pages/SetupWizard.tsx, web/src/pages/System.tsx	Fresh board reaches a useful server-health view; reboot/network-loss recovery is verified.
12 — Hardening and optional cloud sync	Reliability and recovery.	Parser fuzzing, disk-full/crash tests, capacity/soak tests, Neon outbox, batched idempotent upserts and restore.	tests/failure/*, internal/neonsync/*, release workflows	Limits are documented; restore is verified; cloud outage has no impact on monitoring/detection.


Revised milestone cut lines
- Phases 0–4: core differentiating demo — paired Linux/nginx server, secure telemetry stream, local persistence, and live server/route observability.
- Phases 0–6: useful server observability product — server-first incidents and alerts plus supporting external-probe cross-checks.
- Phases 0–10: full functional MVP with evidence-based diagnosis and approval-gated remediation.
- Phases 11–12: appliance polish, hardening and optional Neon backup/synchronization/restore.
The revised HLD/LLD estimates roughly 22–32 weeks of part-time solo effort; actual duration depends on experience, hardware access, testing quality and deployment scope. Treat this only as a planning estimate, not a guarantee.
13. Testing and demo scenarios
The first repeatable demo must prove Mode B. Use a disposable Linux/nginx environment (for example the planned local Docker Compose stack), not a public production service.
Flagship demonstration
1. Start a Linux/nginx test server and generate normal requests.
2. Install and run opsis-agent; verify host samples and normalized route summaries locally.
3. Pair the agent with OPSIS and verify the board-initiated mTLS gRPC stream, heartbeat and reconnect behavior.
4. Confirm CPU, memory, disk, request rate, 4xx/5xx ratio, bytes and p50/p95/p99 appear on the server/route dashboard with correct units and timestamps.
5. Inject a controlled latency or 5xx fault; verify a server-first incident includes the corresponding telemetry and evidence timeline.
6. Stop the Python ML service; demonstrate that Go rules/baselines and the dashboard continue working.
7. Demonstrate a recommendation first. Real actions are added only in Phase 10, on a disposable staging server, with explicit approval, TTL and verification.
Unit tests
- Percentile/t-digest accuracy against known arrays.
- Host metric units, missing-value handling, timestamp/clock-skew behavior and aggregation-window boundaries.
- Baseline warm-up, rolling median/MAD, hysteresis, detection thresholds and incident deduplication.
- Nginx parser behavior for valid, malformed, rotated and adversarial log lines; route normalization and bounded unknown-route labels.
- Schema-version rejection, message-size bounds, sequence numbers, duplicate batches and idempotent handling.
- Redaction, SSRF blocked-destination checks, policy preconditions and exact allowlist enforcement.
Integration tests
- Linux agent sampler and nginx log parser against real demo fixture files.
- Pairing, mTLS certificate pinning/revocation, stream reconnect/resume and duplicate message handling.
- End-to-end route aggregates and host metrics persisted in SQLite and served through REST/SSE.
- Slow, redirecting, unavailable and TLS-expired web targets for supporting probes.
- Python unavailable/slow/bad-schema responses with reliable Go fallback.
- Neon outage only in the late sync phase; local monitoring and detection must remain unaffected.
Security and remediation tests
- Agent connection from an unpaired/revoked certificate is rejected.
- No message can invoke an unregistered action or turn network-provided strings into shell input.
- Unapproved action, stale evidence, failed preconditions, duplicate request and exceeded rate budget are denied.
- Nginx config-test failure restores the previous managed include and does not reload invalid configuration.
- Temporary limits expire after board shutdown or network loss; independent verification catches unsuccessful recovery.
Chaos, soak and demo scenarios
Scenario	Injected condition	Expected behavior
A — Healthy server	Normal nginx traffic and stable host load	Server and routes are healthy; no spurious incident.
B — 5xx/latency fault	Controlled upstream delay or 5xx burst	Server/route metrics change; incident includes time-aligned evidence within the target delay.
C — Resource exhaustion trend	Replay or simulate memory/disk growth	Forecast/warning reflects history and data quality before a hard limit; uncertainty is visible.
D — Traffic flood	Generate synthetic high-rate traffic from a small number of test clients	OPSIS shows request/error/latency evidence; a limit is only applied in the later approved phase and must expire.
E — Agent disconnect	Stop network or terminate the agent	Agent health becomes stale/offline; missing values are not shown as zero; reconnection resumes safely.
F — Board internet loss	Disconnect the appliance network	Dashboard shows board/network degradation; probes do not falsely blame every target.
G — Python outage	Stop the ML service	Go rules and baselines continue; ML status is degraded.
H — Board reboot during temporary limit	Restart/power down the appliance	Server-side expiry still clears the limit; action outcome is auditable.


Report installation/pairing time to first useful view, metric freshness/accuracy, detection delay, false alarms per idle day, precision/recall on injected faults, recovery time, and board CPU/RAM/temperature.
14. Team boundaries
A four-person team can work in parallel after agreeing to shared contracts first.
Workstream	Typical owner	Contract to agree before integration
Board core / Go	Backend lead	Core domain models, REST DTOs, SQLite repository interfaces, both protobuf contracts, metric names/units.
Server agent / Go and Linux deployment	Backend/systems owner	agentlink.proto, agent.yaml, typed action registry, local TTL semantics, supported nginx format and sudoers policy.
Python ML	ML owner	ml.proto, feature schema/version, expected missing-value behavior, model artifact metadata and evaluation report. Start with a mock ML server until the real service is ready.
Console / React	Frontend owner	API DTOs, routes, SSE event payloads, stale-data states, action-approval payload and design tokens.
Hardware, appliance and QA	Device/QA owner	Tested board image/hub, resource budgets, installer behavior, fault-injection scenarios, recovery checks and runbook.


Use small integration PRs and working vertical slices. Do not let each workstream invent a separate route format, metric unit, action schema or feature vector.
15. Definition of done
A feature is not complete just because its source file compiles. It should meet the applicable checks below:
- Correctness: success/failure and boundary behavior are covered by tests.
- Contracts: REST/RPC/DB schema changes are reflected in types and validated with other workstreams.
- Failure behavior: timeouts, cancellation, restart, unavailable dependencies and malformed input fail predictably.
- Security: authorization, input bounds, redaction and action policy are tested for security-relevant paths.
- Resource behavior: concurrency, queues, history ranges and memory use are bounded.
- Observability: useful structured logs/health status exist without secrets or sensitive payloads.
- User experience: loading, empty, stale, offline, unavailable and error states are visible; unsupported metrics are never shown as zero.
- Recovery: actions are verified; temporary changes expire; updates/changes can revert where required.
- Documentation: build/run assumptions, support limitations, and recovery steps are written down.
Project-wide acceptance criteria
1. A fresh install boots into the setup wizard without default credentials.
2. A user can register a supported Linux/nginx server, pair the Go agent, and see host plus per-route telemetry persisted locally across reboot.
3. The first end-to-end demo shows CPU/memory/disk, request rate, 4xx/5xx and p95/p99 from a real test server on the dashboard.
4. Board internet loss and agent disconnect do not create misleading target-down incidents; stale/missing values remain explicit.
5. The dashboard shows data source, freshness, metric coverage and unavailable reasons honestly.
6. mTLS pairing, certificate pinning/revocation, stream reconnect and duplicate-message handling work as specified.
7. Go detection continues if the Python process is stopped or fails its RPC contract.
8. Incidents merge/deduplicate and provide transparent ranked evidence.
9. No disruptive action runs without satisfying the mode, allowlist, precondition, freshness, approval and budget gates.
10. A temporary action expires and post-action recovery is independently verified.
11. Updates are verified and roll back after health-check failure.
12. Optional Neon outages never stop local storage, agent ingestion, incidents or detection. Supporting URL probes also continue where network access permits.
Official learning references
Use official docs first and keep the exact commands in this README aligned with the actual pinned dependency versions.
- Go tutorial: https://go.dev/doc/tutorial/
- Go database access: https://go.dev/doc/tutorial/database-access
- Go net/http: https://pkg.go.dev/net/http
- SQLite documentation: https://www.sqlite.org/docs.html
- modernc.org/sqlite: https://pkg.go.dev/modernc.org/sqlite
- gRPC Go quick start: https://grpc.io/docs/languages/go/quickstart/
- Protocol Buffers tutorial: https://protobuf.dev/getting-started/gotutorial/
- Buf documentation: https://buf.build/docs/
- OpenTelemetry documentation: https://opentelemetry.io/docs/
- scikit-learn Isolation Forest: https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html
- React Learn: https://react.dev/learn
- Vite guide: https://vite.dev/guide/
- TypeScript handbook: https://www.typescriptlang.org/docs/handbook/intro.html
- Recharts: https://recharts.org/en-US/
- systemd service documentation: https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
- Docker Compose documentation: https://docs.docker.com/compose/
- OWASP SSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- Go testing package: https://pkg.go.dev/testing
Final note
Build OPSIS around the Mode B flagship path. First establish the Go control plane and SQLite, then make the Linux/nginx Go agent collect host and route telemetry, connect it through secure pairing and mTLS gRPC, and display real server measurements on the board dashboard. Add supporting external probes for independent cross-checks, then server-first rules, incidents and alerts. Introduce ML in shadow mode only after stable telemetry and deterministic detection exist. Add approval-gated remediation last, with allowlists, persistent expiry, independent verification and rollback proven before expanding its scope.