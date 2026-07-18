# Spec 007: Hermes function-tool bridge

## Objective

Expose a narrowly scoped Hermes function catalog to either voice provider while keeping Hermes as the server-authoritative executor and approval boundary.

## Dependencies

- Specs 002–006

## Deliverables

- Canonical `PortableFunctionTool`/`ToolRequest`/`ToolOutput` models.
- Hermes compatibility adapter for scope, schemas, and dispatch.
- OpenAI and xAI tool serializers/parsers.
- Idempotent tool route and endpoint-neutral coordinator.
- Initial low-risk policy and approval gate.

## Flow

1. Backend resolves exact enabled/disabled Hermes toolsets and allowed names.
2. Backend generates canonical schemas and catalog fingerprint.
3. Provider adapter serializes tools into its session syntax.
4. Adapter maps completed provider calls to normalized requests, including group ID for parallel calls.
5. The active endpoint/provider coordinator submits each call to `/calls/{voice_call_id}/tools/{provider_call_id}` or the equivalent in-process route.
6. Backend revalidates profile, endpoint scope, initiator/participant authority, active call, provider/endpoint/capability/catalog fingerprints, tool scope, arguments, risk, and approval.
7. Hermes dispatcher runs with identical scope/session identifiers.
8. Provider adapter serializes terminal outputs and requests continuation according to provider ordering rules.

Provider-native web/X/file/MCP tools remain disabled initially because they bypass Hermes policy. Only Hermes-backed custom functions are exposed.

Retries keyed by provider session/call ID return the stored result and never execute twice. Parallel provider calls may execute concurrently only when all selected Hermes tools are explicitly parallel-safe; otherwise queue deterministically and send all outputs before one continuation request.

All Hermes-specific imports stay in `hermes_compat.py`; signature/scope resolution failure disables tools rather than dispatching unrestricted.

## Tests

- Canonical schema conversion to both providers and stable fingerprints.
- Unknown/stale/revoked tool, malformed/oversized args, inactive/cross-provider call, timeout, and output bounds.
- Identical Hermes toolset constraints reach dispatcher.
- Idempotent retry and parallel ordering.
- Correct provider output event plus exactly one continuation request.
- Harmless tool end-to-end for each adapter fixture.
- Approval approve/reject/timeout before consequential tools can be enabled.
- Multi-participant tests proving channel presence and unattributed speech cannot inherit initiator tool scope.

## Acceptance criteria

1. Each provider can invoke the same harmless Hermes tool and continue speaking from its result.
2. No provider, renderer, channel adapter, or additional participant can widen the server catalog.
3. Provider-native tools are off by default.
4. Retry cannot duplicate side effects.
5. Hooks, middleware, and proven approvals execute exactly once.

## Non-goals

- all tools enabled by default
- renderer-side execution
- public MCP exposure
- bypassing approval for latency
