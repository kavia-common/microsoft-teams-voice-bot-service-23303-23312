# Architecture — Microsoft Teams Voice Bot Backend (FastAPI)

## Executive Summary and Scope
This document describes the architecture for the “voice_bot_api_backend,” a single-container FastAPI service that integrates with Microsoft Teams to automate call handling using Microsoft Graph Calling APIs and the Microsoft Bot Framework. The current codebase implements a minimal FastAPI application with a health check and CORS middleware. This document defines the target architecture that expands the service to include webhook processing, call control orchestration, administrative APIs, robust security, observability, and operational readiness. The scope includes backend service design only; no UI or persistent database is required in the initial phase.

## Architecture Overview and Context
At a high level, Microsoft Teams emits call-related events and Bot Framework activities to the service’s webhook endpoints. The backend validates authentication, translates events into call control intents, and invokes Microsoft Graph Calling APIs to execute actions such as answer, play prompt, transfer, record, and hang up. The service is designed to be stateless with ephemeral in-memory session state and token caches, allowing horizontal scaling.

Context description:
- Actors: Microsoft Teams services (Graph Calling), Microsoft Bot Framework service, Administrators/Automation clients, Observability backends.
- Ingress: HTTPS requests to FastAPI endpoints for health, webhooks, admin/control.
- Egress: Outbound HTTPS to Microsoft Graph, optional outbound to Bot Framework service URLs, and telemetry exporters (metrics/traces).
- Runtime: Single container named “voice_bot_api_backend”.

## Component and Module Design
The service is composed of the following layers and modules:
- API Layer (FastAPI routers)
  - Public routes: health (implemented), liveness/readiness, webhook receivers for Graph and Bot Framework, admin/control endpoints, optional metrics.
  - Responsibilities: input validation and parsing, authn/authz, correlation ID propagation, request-to-domain mapping.
- Services (Domain orchestration)
  - Call Orchestrator: interprets events, manages ephemeral call session state, determines next action, and orchestrates Graph/Bot operations.
  - Policy/Idempotency Service: ensures safe re-processing of duplicate webhook deliveries and enforces rate limiting/backoff strategies.
- Clients (Integrations)
  - Graph Client: obtains Azure AD app-only tokens (client credentials), calls Graph Calling endpoints with retry/timeout policies.
  - Bot Framework Adapter/Validator: validates Bot Framework JWTs and service URLs, assists with activity processing when used.
- Middleware
  - CORS (implemented), request/response logging, correlation ID injection/propagation, error envelope mapping, and optional OpenTelemetry instrumentation.
- Configuration
  - Environment variable ingestion and validation at startup with feature flags to control optional behaviors (IVR, recording, transcription).

Container/module structure (target layout):
- src/api/main.py: FastAPI app creation, middleware, router registration, health endpoints. (current file)
- src/api/routers/{health,webhooks,admin,calls}.py: Route groups by concern (planned).
- src/services/{orchestrator,state,auth,security}.py: Domain logic.
- src/clients/{graph,bf}.py: Outbound integrations.
- src/middleware/{logging,correlation,authz}.py: Cross-cutting concerns.
- src/config/settings.py: Environment variable parsing and validation.
- src/observability/{logging,metrics,tracing}.py: Telemetry helpers.

## External Integrations
- Microsoft Graph Calling APIs
  - Usage: answer, play prompts (TTS or media), collect input, transfer, record, hang up; subscription and event handling patterns as applicable.
  - Auth: OAuth 2.0 client credentials using Azure AD app-only permissions; tokens cached until expiry.
- Microsoft Bot Framework
  - Usage: receive activities for Teams voice scenarios; validate JWTs, service URLs, and tenant context.
  - Auth: Bot Framework JWT validation per Microsoft guidelines; audience, issuer, timestamp, and service URL checks.

## Authentication and Authorization Flows
- App-only Azure AD authentication
  - The service uses client credentials (tenant ID, client ID, client secret) to obtain tokens for Graph. Tokens are cached and refreshed when near expiry. Outbound requests include Authorization: Bearer <token>.
- Bot Framework JWT validation
  - Incoming Bot activities are validated by verifying token issuers, audiences, signatures, and service URLs. Clock skew and replay protection are applied.
- Webhook authorization for Graph
  - Validate Azure AD-issued tokens on incoming Graph webhook calls. Enforce tenant allow-list and audience/issuer checks.
- Admin API protection
  - Admin endpoints require either a pre-shared bearer token (ADMIN_API_TOKEN) or a JWT from a configured IdP. Only authorized roles can execute call control actions or retrieve session data.

## API Endpoints and Contracts
Current (implemented from code and OpenAPI):
- GET / — Health check. Returns {"message": "Healthy"}.

Planned endpoints (subject to iteration):
- POST /webhooks/graph/calls — Receive and acknowledge Graph calling events. Auth: Azure AD JWT validation. Returns 202/200 quickly to meet SLA.
- POST /webhooks/botframework/activities — Receive Bot Framework activities. Auth: Bot Framework JWT validation.
- POST /admin/test-call — Trigger sandbox test flows. Auth: Admin token/JWT.
- GET /admin/sessions/{callId} — Inspect ephemeral session state/summary. Auth: Admin token/JWT.
- POST /calls/{callId}/actions/{action} — Perform call control action: answer, play, collect, transfer, record, hangup. Auth: Admin token/JWT.

Error response model:
- JSON envelope containing fields such as correlationId, errorCode, message, and optional details, with HTTP status mapping.

## Data Model and In-Memory State Store
- Ephemeral Call Session
  - Fields: callId, tenantId, status (ringing, connected, transferring, recording, terminated), participants (minimal non-PII), currentStep, collected (DTMF/text), correlationId, updatedAt.
  - Storage: per-process in-memory structure keyed by callId; optionally guarded by locks for concurrency.
  - Lifecycle: created on first event, updated on each action, removed on termination/timeouts.
- Token Cache
  - Azure AD app tokens keyed by resource/scope and tenant. Includes expiry metadata for proactive refresh.
- Optional Future Persistence
  - Session summaries and error audits for analysis and compliance reporting.

## Sequence Diagrams (described)
Incoming call (Graph):
- Graph posts a call event to /webhooks/graph/calls with a JWT.
- API validates JWT and tenant; quickly acknowledges.
- Orchestrator loads/creates session, decides action (e.g., answer + greeting).
- Graph Client acquires/uses token, calls Graph to execute actions.
- Session state updated; subsequent events advance flow (collect DTMF, transfer, record).
- On termination, session is removed; logs/metrics/traces include correlationId.

Play prompt:
- Admin or orchestrator triggers “play” on a connected call.
- API maps request to orchestration command.
- Graph Client invokes appropriate Graph action with media/SSML/TTS references.
- Response is recorded; errors mapped to retry/backoff policies.

Transfer:
- Orchestrator determines target (user/queue/resource account).
- Graph Client issues transfer; state moves to transferring then connected/terminated based on result.

Record:
- Orchestrator starts recording when enabled by feature flag and policy.
- Graph Client initiates/controls recording; state tracks recording status.

Hangup:
- Orchestrator issues hangup action through Graph; state moves to terminated and is cleaned up.

Webhook validation:
- Each webhook request includes JWT; API validates audience/issuer/signature/time. Correlation ID is generated or extracted from headers for trace continuity.

## Configuration and Environment Variables
- SERVER_PORT
- LOG_LEVEL
- ALLOWED_ORIGINS
- AZURE_AD_TENANT_ID
- AZURE_AD_CLIENT_ID
- AZURE_AD_CLIENT_SECRET
- BOT_APP_ID
- BOT_APP_PASSWORD
- GRAPH_BASE_URL (default https://graph.microsoft.com)
- BOT_SERVICE_URL
- WEBHOOK_SECRET (optional HMAC for additional verification)
- FEATURE_FLAGS (ivr,recording,transcription)
- METRICS_ENABLED
- TRACE_EXPORTER_URL
- RETENTION_DAYS
- ADMIN_API_TOKEN
- ALLOWED_TENANT_IDS
- REQUEST_TIMEOUT_MS

Configuration validation:
- On startup, validate required variables based on enabled features and fail fast with clear error messages. Non-secret dynamic reload may be considered later.

## Operational Concerns
Deployment:
- Single container deployment of FastAPI app. In Kubernetes, configure liveness (GET /) and add readiness (/health/ready) in future iteration. Set resource requests/limits to expected call volume.

Scaling and availability:
- Stateless processing enables horizontal scaling. Use idempotency for webhook events to prevent duplicate side effects. Consider a shared idempotency store if multiple replicas are deployed.

Rate limiting and backpressure:
- Apply rate limits at the edge. Acknowledge webhooks quickly and process asynchronously if needed. Use upstream-friendly retry policies with exponential backoff and jitter.

Idempotency, retries, and timeouts:
- Include idempotency keys and deduplicate events. Use bounded retries with exponential backoff for transient Graph errors. Set reasonable timeouts (REQUEST_TIMEOUT_MS) for outbound calls.

Reliability:
- Circuit breaking for Graph/Bot calls when error rates spike; implement fallback behaviors (e.g., safe termination with polite message) as policy.

Runbook:
- If 401 on webhooks: verify Azure AD/Bot Framework JWT validation, tenant allow-list, and system clock skew.
- If Graph errors spike: check token acquisition, review rate limits, and status pages; adjust backoff.
- If latency increases: inspect traces for downstream hotspots, DNS/egress health, and token cache.
- For secret rotation: update secret store/environment and trigger a rolling restart.

## Observability: Logging, Metrics, Tracing
- Logging: structured JSON logs with correlationId and callId; redact PII; consistent severity levels; include action outcomes and error codes.
- Metrics: HTTP request rate/latency/error counts; event processing latency; Graph call success/failure rates; token acquisition metrics; active sessions gauge. Expose Prometheus-compatible metrics if enabled.
- Tracing: Use OpenTelemetry instrumentation for inbound routes and outbound Graph/Bot calls. Propagate correlation IDs via headers and include them in span attributes. Export traces to TRACE_EXPORTER_URL.

## Security and Compliance Considerations
- Authentication: Validate JWTs for Graph and Bot Framework; enforce tenant allow-lists and proper audiences/issuers.
- Authorization: Protect admin routes with ADMIN_API_TOKEN or JWT (RBAC preferred). Enforce least privilege for Azure AD app registration.
- Data protection: Minimize and redact PII in logs; encrypt in transit (TLS); do not persist call audio unless explicitly enabled and with lawful basis.
- Anti-replay and freshness: Validate timestamps/nonces; reject stale requests.
- Supply chain: Pin dependencies; scan for vulnerabilities; rotate secrets regularly.
- Compliance: Align with organizational requirements (GDPR/SOC 2). Configure RETENTION_DAYS for logs and ensure data handling policies are followed for recording/transcription features.

## Testing and Verification Strategy
- Unit tests: configuration parsing/validation, JWT validators, token cache behavior, call state transitions, error envelope mapping.
- Integration tests: httpx-based tests against mocked Graph/Bot endpoints; JWT signature validation with test keys; retry/backoff behavior under transient failures.
- End-to-end tests: Use a test tenant or emulator to simulate inbound calls and validate IVR flows, transfer, and hangup.
- Performance tests: Load test webhook endpoints and admin APIs; verify p50/p95 latency SLAs; stress token acquisition and outbound Graph calls.
- Security tests: Negative JWT tests (expired, audience mismatch), replay attempts, rate limit/burst testing, secret scanning and SAST.

## Risks, Assumptions, and Future Enhancements
- Risks: Changes to Graph Calling APIs or Bot Framework authentication flows; webhook delivery variability; clock skew; over-reliance on in-memory state when scaling out.
- Assumptions: Single container deployment; Azure AD and Bot registrations are provisioned; inbound networking from Microsoft services is allowed; no persistent DB required initially.
- Future enhancements: Persistent store for session summaries/audit logs; distributed cache for idempotency across replicas; outbound calling; advanced IVR with NLP; multi-tenant RBAC; multi-region HA; compliance packs (e.g., DLP).

## References to Current Code
- FastAPI application, CORS middleware, and health check are implemented in src/api/main.py.
- The generated OpenAPI describes only the health check endpoint at “/”.

