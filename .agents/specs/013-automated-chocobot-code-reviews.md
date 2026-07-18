# Spec 013: Automated ChocoBot code reviews

## Objective

Run a rigorous ChocoBot review automatically for pull requests in `anton-matosov/hermes-duplex-voice` when a pull request is opened, reopened, or assigned to `anton-matosov`. Receive signed GitHub App webhooks through a stable HTTPS endpoint, isolate the review runtime from Anton's normal Hermes profile, and publish one verified GitHub `COMMENT` review for the exact head commit.

This is repository automation. It must not modify the Duplex Voice runtime, require a GitHub Actions workflow, or let untrusted pull-request content reach Anton's smart-home, personal-memory, or messaging capabilities.

## Dependencies

- [Spec 014: Create the ChocoBot GitHub App](014-create-chocobot-github-app.md) for the final GitHub identity, installation, permissions, webhook secret, and installation-token credentials.
- A domain delegated to Cloudflare DNS for the recommended stable hostname. A different durable HTTPS ingress is acceptable if it preserves the same security properties.
- Hermes Agent with gateway, webhook, profile, and `github-code-review` support.
- Docker available on the Hermes host.

The tunnel, restricted Hermes route, and local fixture tests can be built before the GitHub App exists. Live webhook and review publication tests require the installed app.

## Trigger contract

Accept only a GitHub `pull_request` event for `anton-matosov/hermes-duplex-voice` when:

```text
action == opened
OR action == reopened
OR (action == assigned AND assignee.login == anton-matosov)
```

All other repositories, event types, actions, and assignees fail closed with a successful ignored response and no agent run.

The initial release does **not** trigger on:

- `synchronize` after new commits are pushed;
- `review_requested` when Anton is requested as a reviewer;
- `ready_for_review` when a draft becomes ready;
- comments, labels, checks, pushes, or workflow events.

Those are separate policy changes, not aliases for the requested contract. Opening a draft currently matches `opened`; change that only through an explicit policy decision.

## Responsibility boundary

### Anton must provide or authorize

- Cloudflare account/domain ownership and the chosen public hostname;
- interactive Cloudflare and GitHub account authorization;
- creation and installation of the ChocoBot GitHub App as specified in Spec 014;
- secure placement of the GitHub App private key and confirmation of its path, without pasting it into chat or committing it;
- approval for the first real review posted to a pull request.

### ChocoBot can implement and verify

- rollback backups before each Hermes configuration stage;
- `cloudflared` installation/service configuration after Anton authorizes the tunnel;
- loopback-only Hermes webhook listener and stable route;
- dedicated `prreviewer` profile, minimal tool surface, and Docker-backed review workspace;
- event filtering and payload reduction;
- short-lived GitHub App installation-token broker;
- review prompt, deduplication, atomic publication, read-back verification, logging, tests, and rollback instructions.

The operator sequence is documented in [`docs/automated-code-reviews.md`](../../docs/automated-code-reviews.md).

## Deliverables

The implementation changes host/profile state rather than product code:

```text
Hermes host
├── Cloudflare Tunnel service
├── default gateway webhook listener on 127.0.0.1:8644
├── dynamic route: duplex-voice-pr-review
├── ~/.hermes/scripts/github-duplex-pr-review-filter.py
├── restricted Hermes profile: prreviewer
└── ChocoBot credential broker
    ├── long-lived private key outside model/tool-visible storage
    └── short-lived installation tokens scoped to one installation/repository

GitHub
├── ChocoBot GitHub App
├── installation limited to anton-matosov/hermes-duplex-voice
└── pull_request webhook deliveries
```

No `.github/workflows` file is required.

## Architecture

```text
GitHub App pull_request webhook
        │ HTTPS + X-Hub-Signature-256
        ▼
Cloudflare Tunnel (outbound connector on hermes-box)
        │ http://127.0.0.1:8644
        ▼
Hermes webhook route
        │ event/repo/action filter + reduced payload
        ▼
prreviewer profile
        │ isolated checkout + rigorous review
        │ constrained ChocoBot credential broker
        ▼
GitHub REST: one COMMENT review, then read it back
```

The public URL is profile-qualified:

```text
https://<chosen-hostname>/p/prreviewer/webhooks/duplex-voice-pr-review
```

The webhook listener binds to loopback. No router port-forward or direct public listener is allowed when the tunnel is used.

## Security requirements

### Inbound authenticity

- Require `X-Hub-Signature-256` validation with a strong GitHub App webhook secret.
- Keep SSL verification enabled in GitHub.
- Subscribe the app only to `pull_request` events.
- Preserve Hermes delivery-ID idempotency for GitHub retries.
- Reject malformed JSON, oversized bodies, unknown routes, and invalid signatures before any agent work.

### Untrusted payload handling

- Treat all PR-authored fields and repository contents as untrusted data.
- The route filter emits only repository full name, PR number, action, expected head SHA, base ref, draft state, installation ID, and PR API/HTML identifiers.
- Do not place title, body, comments, commit messages, patch text, or `{__raw__}` in the initial prompt.
- Fetch review material from GitHub only after the route has authenticated and accepted the event.
- PR text cannot override the review contract, request unrelated tools, or change publication policy.

### Profile isolation

- Use a dedicated `prreviewer` profile rather than Anton's default profile.
- Keep approvals in `smart` mode; never use YOLO.
- Use the Docker terminal backend and a fresh temporary checkout per delivery.
- Enable only the toolsets required to read files, run repository checks, load the review skill, and call the constrained GitHub integration.
- Disable Home Assistant, messaging, memory, cron, browser, media generation, unrelated MCP servers, and broad cross-platform delivery.
- Do not mount Anton's home directory, default Hermes profile, SSH keys, git credential store, or unrelated repositories into the review container.

### GitHub App credentials

- Never give the model the GitHub App private key.
- Store the private key outside repository and model/file/terminal-visible roots with owner-only permissions.
- Mint installation tokens on demand. GitHub installation access tokens expire after one hour; do not treat them as permanent PATs.
- Scope installation tokens to the ChocoBot installation and, where supported, the single repository and minimum permissions.
- Prefer a constrained broker/tool that exposes named operations rather than arbitrary authenticated HTTP:
  - read PR metadata, diff, files, reviews, review comments, and checks;
  - create one formal review for a supplied head SHA;
  - read that review and its comments back.
- Do not expose arbitrary repository writes, issue mutation, merges, branch pushes, releases, secrets, administration, or workflow dispatch.
- Redact tokens and credential paths from agent output and logs.

If the installed Hermes version cannot provide a credential broker that keeps the private key outside the agent capability surface, stop and document the missing seam. Do not fall back silently to mounting the private key or Anton's broad personal token into an untrusted review runtime.

## Review workflow

For each accepted event:

1. Resolve the repository and PR number from the reduced payload.
2. Query GitHub and require the PR to remain open and belong to the configured repository.
3. Capture base ref/OID, head ref/OID, draft state, author, description, changed files, checks, and exact diff.
4. Compare the API head SHA with the webhook's expected head SHA. Restart or stop if they differ.
5. Query reviews for a marker matching the ChocoBot identity and current head:

   ```html
   <!-- chocobot-review head:<40-character-sha> -->
   ```

   Skip publication when that marker already exists for the same head. An `opened` event followed by assignment must not post a duplicate review.
6. Create a clean temporary checkout at the exact head and compare against the actual base merge point.
7. Load `github-code-review` and its rigorous PR-review reference.
8. Discover repository and path-specific instructions, then inspect every changed file and hunk with surrounding code and callers.
9. Run independent correctness, security/privacy, testing/reliability, compatibility/operations, and repository-compliance passes.
10. Run repository-defined checks. Record exact commands and outcomes; disclose checks that cannot run.
11. Deduplicate and adversarially validate findings. Published findings require a concrete trigger, observable impact, evidence, smallest practical correction, and confidence at or above the publication threshold.
12. Re-query the PR immediately before publication. If the head changed, discard stale inline locations and review the new head instead of posting against the old one.
13. Publish one atomic formal review with event `COMMENT`, a summary body, the head marker, and validated inline comments.
14. Read the submitted review and comments back. Verify app identity, event, commit ID, body marker, comment count, and URL.
15. Remove the temporary checkout and end the one-shot webhook session.

The initial policy never uses `APPROVE` or `REQUEST_CHANGES`. This avoids impersonating Anton's merge authority and keeps the automation advisory even when it finds a blocker.

## Failure behavior

- Invalid signature or malformed request: reject without agent run.
- Non-matching trigger: return ignored without agent run or token cost.
- Duplicate delivery ID: return duplicate without rerunning.
- Existing marker for current head: record a deduplicated skip.
- PR closed before review: skip.
- Head changes during review: invalidate stale work and retry once against the new head; otherwise report a bounded failure without publishing.
- Partial diff coverage, unavailable checks, or checkout failure: do not claim complete coverage. Publish only if the resulting summary accurately discloses residual risk; otherwise fail without review.
- GitHub API or token broker failure: retry with bounded backoff, then alert through operator logs. Never post a fabricated success result.
- Invalid inline location: preserve other validated findings and convert only the rejected finding to a clearly located summary item if publication remains coherent.
- Model failure or timeout: leave no pending review and expose a diagnosable operator error.

## Observability

Record, without secrets or untrusted bodies:

- GitHub delivery ID, event action, repository, PR number, expected/reviewed head SHA;
- route decision: accepted, ignored reason, duplicate, or failed;
- review start/end time and bounded retry count;
- checks actually run and exit status;
- review ID/URL and read-back result;
- cleanup result.

Provide operator commands for gateway status/logs, tunnel status, route listing, GitHub delivery inspection, pausing the subscription, credential rotation, and rollback.

## Tests

### Filter fixtures

- matching `opened`;
- matching `reopened`;
- matching `assigned` to `anton-matosov`;
- `assigned` to another user;
- `review_requested` for Anton;
- `synchronize`;
- wrong repository;
- wrong event type;
- malformed/missing fields;
- draft `opened` under the chosen draft policy.

### Security and integration

- valid and invalid GitHub HMAC signatures;
- body-size and rate limits;
- reduced payload contains no PR-authored prose;
- private key is unreadable from the review container and agent tools;
- installation token is scoped, redacted, and refreshed after expiry;
- broker rejects non-allowlisted repository/API operations;
- profile cannot access default-profile secrets, Home Assistant, messaging, or host home files;
- exact-head checkout and head-change invalidation;
- duplicate delivery ID and same-head review marker deduplication;
- atomic `COMMENT` review creation and read-back;
- inline comment path/line validation;
- cleanup after success, failure, cancellation, and timeout;
- gateway/tunnel recovery after restart.

### Live smoke

With Anton's explicit approval, trigger one real review on a disposable PR or a nominated open PR. Verify the visible author is `chocobot[bot]` (or the actual slug GitHub assigned), the event is `COMMENT`, the commit ID equals the reviewed head, and replaying another accepted event for that head does not create a second review.

## Acceptance criteria

1. Only the exact trigger contract starts a review.
2. GitHub webhook signatures are required and verified over HTTPS.
3. The review runs in a dedicated Docker-backed profile with no access to Anton's broad default capabilities.
4. The GitHub App private key never enters model context or the review container.
5. ChocoBot reviews the complete diff for a pinned head or explicitly discloses partial coverage.
6. ChocoBot posts one formal `COMMENT` review per head and verifies it by reading it back.
7. Duplicate deliveries and multiple accepted actions for the same head do not duplicate reviews.
8. Review comments are attributed to the ChocoBot GitHub App, not Anton's personal account.
9. Failure paths are bounded, observable, and do not fabricate or half-publish a review.
10. Setup and rollback are documented with a clear Anton/ChocoBot responsibility split.

## Non-goals

- reviewing every push (`synchronize`) in the initial policy;
- treating assignee and requested reviewer as the same GitHub concept;
- approving, requesting changes, merging, pushing fixes, or modifying branches;
- reviewing repositories other than `anton-matosov/hermes-duplex-voice`;
- replacing repository CI;
- exposing Hermes directly to the public Internet without HMAC and TLS;
- granting ChocoBot repository administration, secrets, workflow-write, release, or merge permissions;
- changing Duplex Voice production code.
