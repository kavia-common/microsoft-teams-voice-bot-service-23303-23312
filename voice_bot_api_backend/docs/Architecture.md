# Architecture — Microsoft Teams Voice Bot Backend (FastAPI)

## System Architecture Overview
The “voice_bot_api_backend” is a single-container FastAPI service that integrates with Microsoft Teams via Microsoft Graph Calling APIs and Microsoft Bot Framework. It exposes RESTful endpoints for health, webhooks, and administrative actions. The service validates incoming events, orchestrates call control by invoking Microsoft Graph or Bot Framework capabilities, and emits logs, metrics, and traces for observability. The current implementation in the repository is a minimal FastAPI app with a health check endpoint and CORS enabled; the architecture described here provides the blueprint for expanding to full functionality.

## Component Responsibilities
- API Layer (FastAPI):
  - Public routes: health readiness/liveness, webhook receivers, admin/control endpoints.
  - Request validation, authentication, and authorization.
- Call Orchestrator:
  - Processes call events and maintains ephemeral call state.
  - Executes actions: answer, play prompt, collect input, transfer, record, hang up.
  - Idempotency controls and retry strategy for downstream calls.
- Integrations Layer:
  - Microsoft Graph Client: Handles OAuth token acquisition and REST calls to Graph Calling endpoints.
  - Bot Framework Adapter: Validates and processes Bot activities.
- Security Layer:
  - Token validation (Azure AD, Bot Framework), HMAC/JWT auth for admin routes.
  - Secret management and configuration validation on startup.
- Observability Layer:
  - Structured logging, metrics exposure (optional Prometheus), tracing (OpenTelemetry).
- Configuration:
  - Environment variable ingestion and validation.
  - Feature flags to toggle IVR, recording, and other capabilities.

## Sequence Diagrams (described)
Sequence 1: Inbound Call via Microsoft Graph
- Microsoft Graph sends a call event to /webhooks/graph/calls with JWT.
- API Layer validates token and tenant; enqueues processing (synchronous ack).
- Call Orchestrator loads or creates call session state and determines next action (e.g., answer + prompt).
- Integrations Layer obtains Azure AD token and calls Graph to execute action (answer/play).
- Orchestrator updates state; future events drive additional steps (collect DTMF; transfer).
- Logs/metrics/traces recorded; final termination event clears session state.

Sequence 2: Bot Framework Activity
- Bot Framework posts activity to /webhooks/botframework/activities with signed JWT.
- API Layer verifies signature and service URL.
- Orchestrator maps activity to call control operation (e.g., send TTS or prompt).
- Integrations Layer executes Graph/Bot action as needed.
- Observability recorded end-to-end.

Sequence 3: Admin Session Inspection
- Admin calls GET /admin/sessions/{callId} with admin token.
- API Layer authenticates; Orchestrator reads ephemeral state or persistent summary.
- Response includes status, recent actions, and errors; logs/traces linkable via correlationId.

## Data Models and State Management
Ephemeral Call Session
- callId: unique identifier.
- tenantId: tenant to which the call belongs.
- status: ringing, connected, transferring, recording, terminated.
- participants: caller/callee minimal metadata (non-PII where possible).
- currentStep: IVR step or action pointer.
- collected: DTMF entries and/or recognized text (if enabled).
- correlationId: used across logs and traces.

Token Cache
- Azure AD app tokens cached with expiry.
- Keyed by scope/resource (Graph) and tenant if applicable.

Persistent Storage (optional/future)
- Session summaries, audit logs, and error records for retrospectives and reporting.

## API Specification (Current and Planned)
Current (from repository):
- GET /: Health check returning JSON {"message": "Healthy"}.

Planned endpoints:
- POST /webhooks/graph/calls
- POST /webhooks/botframework/activities
- POST /admin/test-call
- GET /admin/sessions/{callId}
- POST /calls/{callId}/actions/{action}

Authentication:
- Webhooks: Azure AD/Bot Framework JWT validation.
- Admin: HMAC or JWT via ADMIN_API_TOKEN or configured IdP.

## Security Model
- Inbound webhook JWT validation including audience, issuer, tenant, timestamp.
- IP allow-listing for Microsoft service ranges where feasible.
- Admin route protection with token/JWT and role claims.
- Secrets via environment variables and secure secret store.
- PII minimization and log redaction.
- Rate limiting and circuit breaking to mitigate abuse and downstream failures.

## External Integrations
- Microsoft Graph:
  - OAuth 2.0 client credentials to obtain tokens.
  - Calls to answer, transfer, record, play prompt, hang up, and manage subscriptions.
- Microsoft Bot Framework:
  - Activity protocol handling, JWT validation, service URL verification.

## Environment Variables and Configuration
- SERVER_PORT: Container listen port.
- LOG_LEVEL: INFO/DEBUG/WARN/ERROR.
- ALLOWED_ORIGINS: CORS configuration for the API.
- AZURE_AD_TENANT_ID, AZURE_AD_CLIENT_ID, AZURE_AD_CLIENT_SECRET: OAuth for Graph.
- BOT_APP_ID, BOT_APP_PASSWORD: Bot Framework credentials.
- GRAPH_BASE_URL: Default https://graph.microsoft.com.
- BOT_SERVICE_URL: Expected BF service URL for validation.
- WEBHOOK_SECRET: Optional shared secret for additional verification.
- FEATURE_FLAGS: ivr,recording,transcription.
- METRICS_ENABLED: Enable metrics endpoint.
- TRACE_EXPORTER_URL: OpenTelemetry exporter endpoint.
- RETENTION_DAYS: Log retention.
- ADMIN_API_TOKEN: Admin API bearer token if JWT not used.
- ALLOWED_TENANT_IDS: Comma-separated allowlist for tenants.
- REQUEST_TIMEOUT_MS: Default outbound HTTP timeout.

Configuration Validation
- On startup, validate required variables for enabled features; fail fast with descriptive errors.
- Dynamic reload of non-secret config when supported (future).

## Error Handling and Observability
- Error model: HTTP errors with standardized JSON payload including correlationId and errorCode.
- Logging: structured, JSON, PII-safe, correlation via callId/traceId; log levels aligned with severity.
- Metrics: request rate, latency percentiles, error counts, event processing latency, downstream call success/failure.
- Tracing: distributed tracing for webhooks and downstream calls; spans around validation and action execution.
- Alerting: alerts on elevated 5xx, token acquisition failures, webhook validation failures, and increased call terminations.

## Deployment and Runbook
- Containerized FastAPI app, deployed via container orchestration (e.g., Kubernetes) with:
  - Liveness: GET / (current).
  - Readiness: add /health/ready (future).
  - Resource requests/limits sized for expected call volume.
- Steps:
  1) Configure environment variables and secret mounts.
  2) Deploy container image; verify health.
  3) Register webhook endpoints with Microsoft Graph and Bot Framework.
  4) Validate inbound events using test tenants.
- Runbook:
  - If 401 on webhooks: verify JWT validation config and clocks.
  - If Graph 5xx: implement retry/backoff; check circuit breaker status.
  - If high latency: inspect traces, check token cache and DNS/egress health.
  - Secret rotation: update secrets and restart rolling deployment.

## Testing Strategy
- Unit: Call state transitions, validators (JWT, HMAC), config parsing, token caching.
- Integration: Mocked Graph/Bot endpoints with signed tokens; httpx client interactions.
- E2E: Test tenant end-to-end calls; validation of IVR path and transfer actions.
- Performance: Load generation on webhook endpoints; measure latency p50/p95.
- Security: Token misuse tests, replay attacks, invalid signatures, rate limit efficacy.

## Future Enhancements and Phased Rollout
- Phase 0 (current): Health check only (implemented).
- Phase 1: Webhook endpoints, auth, basic IVR, observability baseline.
- Phase 2: Recording/transcription, session summaries, admin controls.
- Phase 3: Outbound calling, richer routing, persistence with RBAC.
- Phase 4: Multi-region HA, advanced analytics, compliance packs.

## Appendix: Operational Considerations
- Idempotency keys for webhook processing to avoid duplicate side effects.
- Clock synchronization required for JWT validation.
- Scaling guidance: stateless event handling enables horizontal scaling; ensure shared idempotency store if needed.
- Backpressure: use queue/buffer or async processing for spikes; quick ACK to webhook sources.
