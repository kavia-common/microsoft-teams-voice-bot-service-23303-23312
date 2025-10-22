# Product Requirements Document (PRD) — Microsoft Teams Voice Bot Backend

## Executive Summary
This document defines the product requirements for the single-container FastAPI backend named “voice_bot_api_backend,” which will power a Microsoft Teams voice bot. The backend will expose RESTful APIs to manage bot interactions, receive and process call events via Microsoft Graph Calling APIs and Microsoft Bot Framework, and orchestrate call flows such as answering, transferring, recording, and performing text-to-speech (TTS) or speech-to-text (STT). The current repository contains a minimal FastAPI application with a health check endpoint; this PRD outlines the goals, scope, and requirements to expand it into a production-ready service.

## Problem Statement and Goals
Microsoft Teams customers need automated voice workflows (e.g., IVR, routing, announcements, recording) to reduce manual handling of high-volume calls and improve responsiveness. The backend should integrate with Microsoft Teams Calling APIs and the Bot Framework to automate call handling and provide reliable, auditable, and scalable execution.

Primary goals:
- Provide a secure, reliable backend to handle Teams call events and orchestrate voice bot actions.
- Expose simple RESTful endpoints for bot control and administrative functions.
- Integrate with Microsoft Graph Calling APIs and Bot Framework webhooks to support inbound/outbound calls, call transfers, TTS/STT interactions, and DTMF input handling.
- Capture observability data for monitoring, troubleshooting, and analytics.

## Non-Goals
- Building custom voice engines; we will leverage Microsoft services for TTS/STT.
- Providing a UI; this phase focuses on backend APIs only.
- Persisting rich customer data; only operational/ephemeral state and minimal configuration are in scope initially.
- Multi-tenant billing/entitlements. Single-tenant configuration with environment variables in early phases.

## Personas and Use Cases
- IT Administrator: Configures credentials, webhook endpoints, and allowed Teams tenants; verifies health and monitors service.
- Contact Center Supervisor: Defines call flows (announce, menu, transfer) and monitors bot performance through observability dashboards.
- Developer/Integrator: Calls admin APIs to trigger test calls, simulate events, and validate integrations.
- End User (Caller): Interacts with the bot in Teams, receives announcements, provides DTMF or speech input, and is routed accordingly.

Key use cases:
- Inbound call handling in Teams: auto-answer, greet, collect input, route/transfer.
- Outbound call initiation (future phase): dial out for notifications.
- Call recording and transcription (where enabled).
- Incident response and monitoring via health and metrics.

## User Stories
- As an IT Admin, I can view a health check endpoint so I know the service is up.
- As a Developer, I can configure environment variables for Azure AD credentials so the bot can authenticate to Graph and Bot Framework.
- As a Supervisor, I can define basic call flows (greeting, menu, routing) that the bot executes.
- As the System, I can receive Microsoft Graph/Bot Framework webhooks to process call state changes.
- As the System, I can produce audit logs and metrics to analyze call volume, drop rate, and response times.
- As an Admin, I can view or retrieve recent errors and traces to troubleshoot issues.

## Success Metrics and KPIs
- Availability: ≥ 99.9% monthly uptime for the API and webhook endpoints.
- Performance: ≤ 200 ms p50 and ≤ 500 ms p95 for API responses (non-callback paths).
- Event Processing Latency: ≤ 1 second p95 from receipt of webhook to action dispatch.
- Error Rate: < 0.5% 5xx responses for API; < 1% failed event handling.
- Integration Reliability: ≥ 99% successful authentication token refreshes.
- Observability Coverage: 100% inbound requests traced; core actions logged with correlation IDs.

## Assumptions and Constraints
- The service runs as a single containerized FastAPI application for this phase.
- Credentials and configuration are provided via environment variables and secure secret stores.
- Microsoft Graph Calling APIs and Microsoft Bot Framework are available and provisioned in the tenant.
- Networking allows inbound HTTPS from Microsoft services to webhook endpoints.
- Persistent storage is optional in MVP; ephemeral in-memory state or cache can be used with safeguards.
- Compliance requirements (GDPR, SOC 2, HIPAA if applicable) influence logging and data retention.

## Functional Requirements
1. Health and Readiness
   - Provide a GET "/" health check returning a simple JSON response (exists).
   - Provide readiness and liveness endpoints suitable for container orchestration.

2. Authentication and Authorization
   - Support OAuth 2.0 client credentials with Azure AD for Microsoft Graph.
   - Validate Bot Framework authentication tokens on incoming requests (signed JWT).
   - Support HMAC or JWT-based auth on admin endpoints (configurable).

3. Webhook Endpoints
   - Receive and validate Microsoft Graph Calling and Bot Framework events (call started, connected, DTMF received, recording state, terminated).
   - Idempotent processing to handle potential retries.
   - Respond with appropriate status codes within SLA.

4. Call Control Actions
   - Answer call, play prompt (TTS or audio file), collect input (DTMF/speech), transfer to user/queue, record, hang up.
   - Provide server-side orchestration of simple IVR menus.
   - Maintain call state and correlation IDs.

5. Admin and Control APIs
   - Trigger test call flows in sandbox mode (non-production tenants).
   - Retrieve recent call sessions metadata, status, and errors.
   - Rotate credentials and force token refresh endpoints (secured).

6. Observability
   - Structured logging with correlation IDs and PII-safe policies.
   - Metrics: request rate, error rate, latency, call success/fail counters, event processing latencies.
   - Tracing spans for inbound/outbound calls and Graph/Bot SDK actions.

7. Configuration Management
   - Read environment variables for all external endpoints, OAuth settings, webhook secrets, and feature flags.
   - Validate configuration at startup; fail-fast with clear errors.

## Non-Functional Requirements
- Performance: As above; scale within container constraints; optimize network calls and caching of tokens.
- Scalability: Horizontal scaling by running multiple container replicas; stateless processing where possible; design for idempotency.
- Reliability: At-least-once processing for webhooks; internal retry logic for transient Graph errors; circuit breakers for downstream.
- Security: Principle of least privilege; secure secret handling; TLS termination; input validation; anti-replay on webhooks.
- Compliance: Log minimization; PII redaction; retention controls; lawful basis for recording/transcription.
- Availability: Kubernetes/Container orchestration readiness/liveness; zero-downtime deploys.
- Maintainability: Clear API contracts via OpenAPI; code linting, tests, and CI integration.
- Compatibility: Microsoft Graph Calling APIs, Bot Framework protocol versions as configured.

## API Specification (Routes, Schemas, Auth)
Current implemented:
- GET /: Health check returning {"message":"Healthy"}.

Planned examples (subject to iteration):
- POST /webhooks/graph/calls: Receive call event from Graph. Auth: Azure AD JWT validation.
- POST /webhooks/botframework/activities: Receive Bot Framework activity. Auth: Bot Framework JWT validation.
- POST /admin/test-call: Trigger a sandbox test call. Auth: Admin token/JWT/HMAC.
- GET /admin/sessions/{callId}: Fetch call session metadata. Auth: Admin token.
- POST /calls/{callId}/actions/{action}: Perform action (answer, play, collect, transfer, record, hangup). Auth: Admin token or internal.

Schemas (illustrative):
- CallEvent: { callId, tenantId, eventType, timestamp, payload }
- ActionRequest: { action, parameters, correlationId }
- SessionSummary: { callId, status, startedAt, endedAt, actions[], errors[] }

## Data Models and State Management
- Ephemeral call session state keyed by callId with:
  - status, participants, current step, collected input, correlationId.
- Token cache for Azure AD app tokens.
- Minimal persistent storage optional in MVP; if introduced, a lightweight store for session summaries and error logs.

## Security Model and Threat Considerations
- Authentication:
  - Validate Bot Framework and Graph JWTs against tenant and audience.
  - Admin endpoints protected via pre-shared HMAC or JWT issued by internal IdP.
- Authorization:
  - Restrict by tenantId and allowed application IDs.
- Threats and mitigations:
  - Replay attacks: include timestamp, nonce; reject stale messages.
  - Forged requests: strict JWT validation and IP allow-listing (if feasible).
  - PII leakage: redact logs; store minimal data; encryption in transit.
  - Secret exposure: use environment variables and secret stores; rotate regularly.
  - DoS: rate limiting and circuit breakers.

## External Integrations (Microsoft Graph, Bot Framework)
- Microsoft Graph Calling: Outbound calls for actions (answer, transfer, record), incoming call events via subscriptions/webhooks.
- Bot Framework: Activities for voice-enabled bots, authentication, and message flow for prompts and recognition.
- Azure AD: OAuth 2.0 client credentials for obtaining tokens to call Graph APIs.

## Environment Variables and Configuration
- SERVER_PORT: Port the FastAPI app listens on.
- LOG_LEVEL: Logging level (INFO/DEBUG/WARN/ERROR).
- ALLOWED_ORIGINS: CORS allowed origins list.
- AZURE_AD_TENANT_ID: Tenant ID for Azure AD.
- AZURE_AD_CLIENT_ID: Application (client) ID for Graph.
- AZURE_AD_CLIENT_SECRET: Client secret for Graph.
- BOT_APP_ID: Bot Framework app ID.
- BOT_APP_PASSWORD: Bot Framework app password/secret.
- GRAPH_BASE_URL: Base URL for Microsoft Graph (default https://graph.microsoft.com).
- BOT_SERVICE_URL: Expected Bot Framework service URL (for validation).
- WEBHOOK_SECRET: Optional HMAC secret for additional webhook verification.
- FEATURE_FLAGS: Comma-separated feature toggles (e.g., ivr,recording).
- METRICS_ENABLED: Enable Prometheus/metrics export (true/false).
- TRACE_EXPORTER_URL: OpenTelemetry exporter endpoint.
- RETENTION_DAYS: Log retention policy days.
- ADMIN_API_TOKEN: Token for securing admin endpoints (if not using JWT).
- ALLOWED_TENANT_IDS: Comma-separated list of allowed tenants.
- REQUEST_TIMEOUT_MS: Outbound Graph HTTP timeout.

## Error Handling and Observability
- Standardized error envelope with correlationId and errorCode.
- Idempotency keys for event processing to avoid duplicate actions.
- Logging: structured JSON logs, no sensitive data, correlation via callId and traceId.
- Metrics: HTTP latency, error rate; call action counters; Graph call latencies.
- Tracing: Spans for webhook receipt, validation, downstream Graph calls.

## Deployment and Runbook
- Container image built from FastAPI app; environment variables injected at runtime.
- Health checks: GET / (liveness) and dedicated /health/ready (readiness) in future iteration.
- Rolling deployments; monitor 5xx rate and latency; canary on 10% of traffic if supported.
- Runbook:
  - Validate environment variables set; test GET /.
  - Confirm webhook endpoints reachable from Microsoft services.
  - Rotate secrets on schedule; verify token refresh logs.
  - On incident, gather logs by correlationId, inspect metrics dashboards, and test Graph connectivity.

## Testing Strategy (Unit, Integration, E2E, Manual)
- Unit Tests: Input validation, token caching, call state machine transitions.
- Integration Tests: Webhook signature/JWT validation against mocked Microsoft tokens; calling Graph sandbox endpoints via httpx with test doubles.
- E2E Tests: Simulate inbound call events through Bot Framework emulator or test tenant; verify full flow.
- Performance Tests: Load test webhook endpoints and admin APIs.
- Security Tests: Static analysis, secret scanning, negative JWT tests, rate limiting tests.

## Future Enhancements and Phased Rollout
- Phase 0 (Current): Minimal app with health check.
- Phase 1: Webhook endpoints, simple IVR actions, Azure AD auth, observability baseline.
- Phase 2: Call recording and transcription, session summaries, admin APIs for sessions.
- Phase 3: Outbound call campaigns, sophisticated routing, multi-tenant configuration, dashboards.
- Phase 4: High availability across regions, advanced analytics, compliance extensions (e.g., DLP).

## References
- Current codebase: FastAPI app with CORS and health check.
- Microsoft Graph Calling APIs documentation.
- Microsoft Bot Framework documentation.
