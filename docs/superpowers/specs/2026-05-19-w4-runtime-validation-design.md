# W4 Runtime Validation Foundation Design

## Status

- Date: 2026-05-19
- Scope: `apps/daemon` first validation slice
- Target: `/api/chat` and `/api/proxy/*/stream`
- Decision: approved

## Purpose

This design defines the first W4 slice from `specs/current/maintainability-roadmap.md`.

The goal is to add a reusable runtime validation foundation at the daemon boundary without prematurely doing W5 modularization. The first slice focuses on the highest-risk HTTP entry points:

- `POST /api/chat`
- `POST /api/proxy/anthropic/stream`
- `POST /api/proxy/openai/stream`
- `POST /api/proxy/azure/stream`
- `POST /api/proxy/google/stream`

## OPM Frame

### Objects

- **Contract Schemas**
  - Runtime schemas defined in `packages/contracts`
  - Own request-shape truth for `chat` and `proxy`
- **Daemon Validation Layer**
  - Thin helper layer in `apps/daemon`
  - Parses raw route input and converts schema failures into shared API errors
- **Route Handlers**
  - Existing `server.ts` route bodies
  - Remain responsible for orchestration and provider-specific behavior
- **Security Checks**
  - Local capability rules such as loopback/origin checks and private-network blocking
  - Stay in the daemon, not in contracts
- **Shared Validation Error Envelope**
  - `ApiError` with `code: VALIDATION_FAILED`
  - `details.kind = validation`
  - `details.issues = [...]`

### Processes

1. A route receives raw `req.body`, `req.params`, or `req.query`.
2. The daemon validation layer parses that input with a contracts schema.
3. If parsing fails, the route returns a shared validation envelope.
4. If parsing succeeds, the route runs daemon-only security checks.
5. If a security check fails, the route returns explicit `FORBIDDEN` or `BAD_REQUEST`.
6. Only validated and permitted input reaches the existing chat or proxy execution flow.

### OPL Statements

- Contract Schemas validate request shape.
- Daemon Validation Layer translates schema failure into `VALIDATION_FAILED`.
- Route Handlers execute business flow only after validation succeeds.
- Security Checks constrain local execution even when request shape is valid.
- This slice changes request validation behavior but does not change SSE success semantics.

## Architectural Decision

### Recommended approach

Use a **contracts-first validator layer**.

Runtime schemas are added beside existing API types in `packages/contracts`, and daemon routes consume those schemas through a small local helper. This keeps compile-time types and runtime validation aligned and creates a reusable base for later W5 modularization and W8 test expansion.

### Rejected alternatives

#### Daemon-local schemas only

This would be faster to start but would keep contracts and runtime validation split across packages.

#### Thin handwritten guards only

This would reduce immediate risk but would not create a real validation foundation and would likely be replaced in the next slice.

## File Design

### `packages/contracts/src/api/chat.ts`

Add runtime schemas for:

- `ChatRequest`
- `ChatCommentAttachment`
- nested comment attachment position/member structures if required

Keep the existing interfaces. The runtime schemas become the source for daemon parsing, not a replacement for the exported types.

### `packages/contracts/src/api/proxy.ts`

Add runtime schemas for:

- `ProxyMessage`
- `ProxyStreamRequest`

The schema must cover:

- `baseUrl`
- `apiKey`
- `model`
- `systemPrompt`
- `messages`
- `maxTokens`
- `apiVersion`

### `packages/contracts/src/errors.ts`

Reuse:

- `ApiValidationIssue`
- `ApiValidationErrorDetails`
- `ApiError`

If needed, add one small helper that maps runtime-schema issues into the shared `issues[]` shape.

### `apps/daemon/src/server.ts`

Keep routes in place for this slice.

Replace scattered route-local input guards in `chat` and proxy routes with calls into a thin validation helper. Avoid opportunistic modularization here; W5 owns route extraction.

### `apps/daemon/src/validation.ts` or equivalent focused helper

Introduce one small helper module whose only responsibilities are:

- parse raw input with a provided schema
- map issues into the shared error detail format
- send `VALIDATION_FAILED` in a consistent response shape

This helper must not absorb route business logic, provider behavior, SSE behavior, or daemon-only authorization rules.

### `apps/daemon/tests/chat-route.test.ts`

Extend route tests to cover:

- missing required fields
- empty-string required fields
- invalid field types
- invalid nested comment attachment data
- assertion of shared validation envelope

### `apps/daemon/tests/proxy-routes.test.ts`

Extend route tests to cover:

- missing required fields
- invalid `baseUrl`
- invalid or empty `messages`
- invalid message role/content shapes
- assertion of shared validation envelope

## Scope

### In scope

- reusable validation helper for daemon HTTP boundary parsing
- contracts runtime schemas for `chat` and `proxy`
- shared validation error envelope for the targeted routes
- route tests proving failure shape and success preservation

### Out of scope

- upload, files, deploy, media, config, and other daemon routes
- `server.ts` route extraction or service decomposition
- SSE event-contract redesign
- provider-specific semantic validation beyond basic field shape and range
- broader cross-platform hardening beyond existing checks

## Validation Rules

### Chat route

`POST /api/chat` must require:

- `agentId`: non-empty string
- `message`: non-empty string

Optional fields may be absent or explicitly `null` where the current contract allows nullability, but when provided they must conform to their expected string or array shape.

`commentAttachments`, when present, must validate nested fields rather than relying on silent normalization.

### Proxy routes

All proxy routes must validate common request structure first.

Common requirements:

- `apiKey`: non-empty string, except only existing route behavior may decide defaults elsewhere
- `apiKey`: non-empty string
- `model`: non-empty string
- `messages`: array of valid `{ role, content }` items

Provider notes:

- Anthropic/OpenAI/Azure require `baseUrl`
- Google may omit `baseUrl`, in which case the existing public default URL is applied before daemon security checks
- `maxTokens` receives only basic numeric validation in this slice
- `apiVersion` receives basic string validation in this slice

### Security boundary rule

Base URL network policy remains a daemon concern.

Schema validation answers **"is the shape valid?"**

Daemon security answers **"may this valid request access this target?"**

## Error Model

### Validation failure

Validation failures return:

- HTTP status: `400`
- `error.code = VALIDATION_FAILED`
- `error.message = Invalid request payload` or a similarly stable shared message
- `error.details.kind = validation`
- `error.details.issues = [...]`

Each issue must include a stable `path` and human-readable `message`. A schema issue may also include a normalized `code`.

### Security failure

Security failures continue to use explicit route-level codes, for example:

- `FORBIDDEN` for private-network or local-origin rejection
- `BAD_REQUEST` for provider-specific request problems that are not generic schema failures

### Upstream/provider failure

Proxy SSE error behavior stays unchanged in this slice. Validation happens before the upstream call path starts.

## Testing Strategy

### Route-level proof

Primary proof lives in daemon route tests because this slice protects HTTP boundaries.

Required assertions:

1. invalid input returns `400`
2. error shape matches shared `ApiError`
3. `error.code` is `VALIDATION_FAILED`
4. `details.kind` is `validation`
5. `details.issues` includes stable field paths

### Behavior-preservation proof

Existing happy-path tests for `chat` and `proxy` must remain green to prove the slice adds guardrails without changing successful streaming behavior.

### Verification commands

- `pnpm --filter @open-design/daemon test`
- `pnpm --filter @open-design/daemon typecheck`
- `pnpm guard`
- `pnpm typecheck`

## Success Criteria

- `chat` and proxy stream routes stop relying on scattered ad hoc input checks
- targeted routes return one shared validation envelope for malformed input
- daemon-only security checks remain explicit and separate from schema parsing
- the new schemas are reusable by later W4 slices and W5/W8 work

## Risks and Guards

### Risk: contracts schemas drift from existing route behavior

Guard: add route tests for current accepted valid payloads before tightening optionality.

### Risk: route behavior changes accidentally while adding validation

Guard: preserve existing success-path tests and limit changes to boundary parsing plus error mapping.

### Risk: helper grows into premature route framework

Guard: keep helper responsibilities narrow and leave route decomposition to W5.

## Implementation Handoff

The next planning phase should create a bounded implementation plan that:

1. adds contracts schemas
2. adds one daemon validation helper
3. migrates `chat` and proxy routes to that helper
4. expands daemon tests
5. verifies no regression in current successful flows
