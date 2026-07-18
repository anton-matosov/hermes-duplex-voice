# Spec 006: Hermes function-tool bridge

## Objective

Let OpenAI Realtime invoke one explicitly configured harmless read-only Hermes tool while Hermes remains the server-authoritative execution and approval boundary.

## Dependencies

- Specs 002, 004, and the v0.1 release in Spec 005.

## Host-seam prerequisite

The current dashboard `plugin_api.py` route receives no `PluginContext`, active agent, parent tool policy, session object, or approval identity. `ctx.dispatch_tool()` is not available inside that independently imported module, and Hermes exposes no public plugin API for executing an externally requested tool with full agent/approval context.

Before implementing the bridge, choose and verify one path:

1. **Preferred:** add or consume a generic upstream Hermes server-side tool-execution seam that accepts an authenticated profile/session context and preserves canonical scope, hooks, middleware, and approvals.
2. **Limited compatibility path:** isolate version-pinned internal Hermes imports in `hermes_compat.py`, contract-test their signatures and exact scope propagation, and expose only the harmless read-only tool. This path cannot claim approval support.

If neither path can prove exact scope and execution behavior, tools remain disabled and Track A may release voice without Spec 005.

## Deliverables

- A verified supported or version-pinned execution seam documented with compatible Hermes versions.
- `hermes_compat.py` containing all non-public Hermes imports, if the compatibility path is used.
- One configured allowlisted tool and its OpenAI function schema.
- Backend route `POST /calls/{call_id}/tools/{tool_call_id}`.
- OpenAI function-call parser and function-output continuation.
- Idempotent result store bounded by call lifetime.
- End-to-end harmless tool smoke.

## Flow

1. Backend resolves the exact allowlisted tool for the active Hermes profile/session.
2. Backend returns the schema as part of authoritative OpenAI session configuration.
3. OpenAI emits final function arguments.
4. Desktop submits call ID, provider call ID, name, and bounded arguments to the plugin route.
5. Backend rechecks active call, exact tool name/scope, arguments, timeout, and output limit through the verified execution seam.
6. The verified seam dispatches through Hermes with the tested profile/session and middleware context; no behavior is inferred merely because the HTTP route is authenticated.
7. Duplicate provider call IDs return the stored terminal result without executing again.
8. Desktop sends `function_call_output` with the same call ID and one `response.create` continuation.

A missing/failed scope lookup means no tools. Renderer schemas are never authorization. Provider-native tools remain disabled.

## Approval boundary

The first release excludes approval-requiring, destructive, or side-effecting tools. `plugin_api.py` authentication is not an approval context. Approve/reject/timeout support requires a new or verified upstream Hermes execution/approval seam and an end-to-end test through Hermes's native approval surface before any such tool is enabled.

## Tests

- Tool schema generated from the real Hermes compatibility seam.
- Unknown/revoked tool, malformed/oversized arguments, inactive/cross-profile call, timeout, and oversized output.
- Identical enabled scope reaches schema generation and dispatch.
- Duplicate/reordered provider calls execute at most once.
- Exactly one OpenAI output item and continuation for a terminal result.
- Plugin hot unload/call close cancels or safely finalizes in-flight calls.
- Harmless live smoke speaks a result.

## Acceptance criteria

1. OpenAI can call one harmless Hermes tool and continue speaking.
2. Renderer or model cannot widen the backend catalog.
3. Retry cannot execute the tool twice.
4. Scope/signature/context incompatibility disables tools and leaves voice conversation usable.
5. No consequential tool is available, and no claim of native approval support is made without a public/verified execution context seam.

## Non-goals

- all Hermes tools;
- renderer-side execution;
- public MCP exposure;
- parallel tool execution;
- channel-participant authorization;
- approvals for side effects.
