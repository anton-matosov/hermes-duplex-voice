# Spec 005: Hermes function-tool bridge

## Objective

Allow the OpenAI Realtime model to call a narrowly scoped set of Hermes tools while preserving Hermes's toolset restrictions, middleware, hooks, guardrails, approvals, and result limits.

## Dependencies

- Spec 002 call registry
- Spec 004 function-call domain events

## Deliverables

- Hermes compatibility adapter for toolset resolution, schema listing, and dispatch.
- Realtime schema conversion and catalog fingerprinting.
- Authenticated, idempotent tool-call route.
- Desktop function-call coordinator.
- Initial low-risk tool policy and explicit approval limitations.

## Compatibility adapter

All Hermes-specific imports live in `backend/.../compat.py`. Against the initially supported Hermes version, the adapter should use the same effective platform toolset configuration as Desktop/CLI and call the canonical equivalents of:

- `model_tools.get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode=True)`; and
- `model_tools.handle_function_call(..., enabled_tools=..., enabled_toolsets=..., disabled_toolsets=...)`.

Implementation must verify current signatures at startup. It must never call the dispatcher with `enabled_toolsets=None` merely because scope resolution failed; that would mean unrestricted access. Scope-resolution failure disables tools and marks `/capabilities` incompatible.

## Tool policy

The voice call has a server-authoritative policy containing:

- effective enabled and disabled toolsets;
- exact allowed tool names;
- schema catalog fingerprint;
- per-tool risk class;
- concurrency and timeout limits; and
- whether a working approval round-trip exists.

Initial release policy:

- deny agent-loop-only tools;
- deny tools unavailable to the associated Hermes platform/profile;
- deny destructive or approval-requiring tools until native approval UI is proven;
- prefer read-only/search/status tools;
- cap concurrent calls at one unless a tool is explicitly safe for parallel execution; and
- cap result size before returning it to the Realtime conversation.

The desktop cannot widen this policy. A client-supplied catalog is never authoritative.

## Schema conversion

Hermes returns OpenAI-format function definitions. Convert these into GA Realtime function tools while preserving:

- function name;
- user-facing description;
- JSON Schema parameters; and
- required fields.

Reject unsupported recursive/excessively large schemas and report omitted tool names diagnostically. Do not silently loosen parameter validation.

## Execution route

`POST /calls/{voice_call_id}/tools/{realtime_call_id}`:

```json
{
  "request_id": "uuid",
  "catalog_fingerprint": "sha256:...",
  "name": "web_search",
  "arguments": {"query": "..."}
}
```

Backend sequence:

1. Authenticate and bind the request to the voice call/profile.
2. Verify call is active and model call ID has not completed.
3. Verify catalog fingerprint and exact tool name.
4. Validate argument JSON, depth, and size.
5. Re-resolve current Hermes tool scope and reject if access was revoked.
6. Run risk/approval policy.
7. Dispatch through Hermes with voice session/task/tool IDs and identical toolset restrictions.
8. Apply timeout and output-size policy.
9. Store the idempotent terminal result.
10. Return a JSON-string-compatible output plus status metadata.

Retries with the same request and Realtime call ID return the stored result. A conflicting retry returns `409` and never executes twice.

## Desktop coordinator

On a completed Realtime function call:

- parse arguments exactly once;
- transition UI to `tool_running` with a safe tool label;
- invoke the backend route;
- create a Realtime `function_call_output` item with the same model `call_id`;
- send `response.create` after output is accepted by the data channel;
- return denials/errors as structured tool outputs so the model can explain or choose another action; and
- never execute tools in the renderer.

Do not speak raw JSON or sensitive arguments. Spoken filler is controlled by session instructions and UI state, not a second model.

## Approval milestone

Approval-requiring tools stay disabled until an integration test proves:

1. Hermes emits its normal approval request for the voice session.
2. Desktop renders the native approval UI.
3. Approve and reject both unblock the backend request correctly.
4. Timeout fails closed.
5. Realtime receives only the terminal approved/denied result.

If the current plugin API cannot attach to Hermes's approval callback, document the missing upstream seam and ship without those tools.

## Tests

- Tool scope resolution and startup signature probes.
- Schema conversion fixtures and catalog fingerprint stability.
- Deny on missing scope, stale fingerprint, unknown tool, malformed args, oversized args, inactive call, or revoked access.
- Dispatcher receives identical enabled/disabled toolsets and session identifiers.
- Idempotent retry never double-executes.
- Timeout and output truncation.
- Desktop emits correct `function_call_output` and exactly one `response.create`.
- End-to-end harmless tool integration in temporary `HERMES_HOME`.
- Approval integration tests before enabling consequential tools.

## Acceptance criteria

1. Realtime can invoke at least one harmless Hermes tool and speak from its result.
2. A tool outside the server catalog cannot execute even if requested manually over REST.
3. Tool Search cannot discover or invoke tools outside the voice session's scope.
4. Hooks, middleware, and supported approval policy fire once per execution.
5. Retry cannot duplicate side effects.

## Non-goals

- exposing all Hermes tools by default
- client-side tool execution
- public remote MCP exposure
- bypassing approvals for lower latency
