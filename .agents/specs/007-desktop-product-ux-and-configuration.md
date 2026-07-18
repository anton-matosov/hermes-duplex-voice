# Spec 007: Desktop product UX and configuration

## Objective

Turn the working voice runtime into a coherent Hermes Desktop feature with native interaction patterns, accessible controls, configuration, diagnostics, and remote-backend support.

## Dependencies

- Specs 001–006

## Deliverables

- Composer action, call panel, and command-palette/keybind contributions.
- Model/voice/device/turn-detection settings.
- Native approval integration where available.
- Accessible call controls and diagnostics view.
- Complete setup, recovery, and uninstall documentation.

## Entry points

Register through the desktop plugin SDK:

- a `COMPOSER_AREAS.actions` call button;
- a draggable call panel or dialog that remains visible while navigating;
- command-palette actions for start, mute/unmute, and end call; and
- optional rebindable key actions.

Do not replace or monkey-patch the built-in microphone button. Label the feature clearly as Realtime/Duplex Voice so users can distinguish it from dictation.

## Call panel

Minimum controls:

- start/end call;
- mute/unmute microphone;
- selected input device;
- connection/listening/speaking/tool status;
- live transcript with user/assistant distinction;
- elapsed time;
- active model and voice; and
- expandable diagnostics/recovery action.

Use Hermes UI components and theme variables. Do not hardcode colors. Hot reload/unload must end active calls safely.

## Settings

Expose only supported backend configuration:

- enabled toggle;
- Realtime model;
- output voice (change requires a new call once audio has started);
- semantic/server VAD choice and supported thresholds;
- maximum call duration;
- allowed voice toolsets or curated tool policy;
- input device preference stored locally per device; and
- transcript persistence and optional post-call summary toggles.

Secrets are configured through Hermes credential workflows and are never editable as plain text in the runtime plugin.

## Remote backend behavior

Hermes may run on another machine while the desktop owns the microphone. The feature must:

- use local renderer WebRTC/media APIs;
- call the selected profile's authenticated remote plugin backend through `ctx.rest()`;
- receive the ephemeral secret from that backend;
- keep direct Desktop ↔ OpenAI media after bootstrap;
- clearly distinguish backend connectivity from Realtime connectivity; and
- avoid plugin WebSocket dependencies that do not work under OAuth remotes unless there is a polling/REST fallback.

## Approvals

When Spec 005's approval milestone is supported:

- show the native Hermes approval request rather than a custom misleading clone;
- keep model audio from claiming success before approval and execution complete;
- allow explicit approve/reject and timeout;
- announce only concise status, never secret arguments; and
- resume the Realtime response with the terminal tool output.

Without a proven approval path, hide approval-requiring tools from configuration.

## Accessibility

- All actions keyboard reachable with visible focus.
- Every status has text, not color alone.
- Transcript supports screen readers without announcing every delta repeatedly; announce finalized turns.
- Mute and call status use appropriate pressed/live-region semantics.
- Respect reduced motion.
- Do not auto-start microphone capture on app launch or plugin load.
- Permission requests occur only after an explicit user gesture.

## Diagnostics

Provide a copyable, redacted diagnostic report containing:

- plugin and Hermes versions;
- capability probe results;
- connection-state transitions and timings;
- browser media capability and selected device kind (not label unless user opts in);
- OpenAI error category and request IDs where safe;
- tool catalog fingerprint and counts, not schemas/arguments; and
- backend/local/remote mode.

Never include transcript text, API/client secrets, SDP, authorization headers, raw tool args/results, or raw audio.

## Tests

- Component interaction and accessibility tests.
- Keyboard and command-palette actions.
- Settings validation and voice-change-new-call behavior.
- Profile switch and remote-backend routing.
- Approval available/unavailable feature gating.
- Diagnostic redaction snapshots.
- Hot unload, app navigation, and window-close cleanup.
- Manual cross-platform checks on Linux, macOS, and Windows.

## Acceptance criteria

1. Users can discover, start, control, and end Duplex Voice without a terminal.
2. The UI is clearly distinct from dictation and reports truthful state.
3. Permission and device failures have actionable recovery.
4. Remote Hermes backends work without routing audio through the backend.
5. Accessibility checks pass and diagnostics contain no sensitive content.

## Non-goals

- redesigning Hermes Desktop's core composer
- mobile-native clients
- SIP/phone UI
- custom visualizer effects that delay core usability
