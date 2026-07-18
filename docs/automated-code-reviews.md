# Automated code reviews with ChocoBot

This guide sets up automatic ChocoBot reviews for pull requests in `anton-matosov/hermes-duplex-voice`. It deliberately separates the steps Anton must perform in account-owned web interfaces from the host configuration and verification ChocoBot can perform.

> **Status:** design and operator guide only. The tunnel, GitHub App, Hermes profile, webhook route, and credential broker do not exist yet.

The implementation requirements live in:

- [Spec 013: Automated ChocoBot code reviews](../.agents/specs/013-automated-chocobot-code-reviews.md)
- [Spec 014: Create the ChocoBot GitHub App](../.agents/specs/014-create-chocobot-github-app.md)

## What the finished system does

GitHub sends a signed `pull_request` webhook to Hermes when repository activity occurs. Hermes starts a review only when the PR is:

- opened;
- reopened; or
- assigned to `anton-matosov`.

ChocoBot reviews the exact head commit in an isolated profile, posts one formal GitHub `COMMENT` review, and reads it back to verify publication. If opening and assigning produce two webhook deliveries for the same head, the second run is deduplicated.

The first version does not automatically review new pushes (`synchronize`), reviewer requests (`review_requested`), or draft-to-ready transitions (`ready_for_review`). Opening a draft does match `opened` unless the policy is changed before implementation.

## Cost

Cloudflare Tunnel is available on Cloudflare's free plan. A stable named hostname normally requires a domain using Cloudflare DNS; the domain itself may have a registration cost. Do not use a random TryCloudflare Quick Tunnel for durable automation because Cloudflare treats it as development-only and does not guarantee its URL or availability.

GitHub Apps do not require a paid GitHub plan for this repository setup. Model/API usage for each review still follows the configured Hermes provider's billing or subscription.

## Responsibility matrix

| Area | Anton | ChocoBot |
|---|---|---|
| Cloudflare account and domain | Own account, choose domain/hostname, authorize tunnel | Validate prerequisites, install/configure connector after authorization |
| GitHub App | Create and install through GitHub UI | Supply exact settings, verify permissions/events/installation |
| Secrets | Download/place private key and enter webhook secret in GitHub | Generate secret securely, lock down files, configure broker without exposing values |
| Hermes | Approve configuration scope and first live review | Back up, create profile, enable webhook, add filter/subscription, test and verify |
| Public review | Choose disposable/nominated PR and authorize first write | Review pinned head, publish one `COMMENT` review, read it back |
| Ongoing operations | Approve permission changes and rotate account-owned credentials | Monitor, diagnose, rotate host configuration, and execute rollback when requested |

## Before starting

Anton decides and records:

- the Cloudflare-managed domain;
- the hostname, for example `chocobot-hooks.example.com`;
- whether drafts should be reviewed when opened;
- whether later pushes should add the `synchronize` trigger;
- which PR may receive the first real ChocoBot review.

ChocoBot verifies:

- the local repository and default branch;
- Hermes version and gateway health;
- Docker availability;
- that port `8644` is free;
- that no existing webhook/tunnel/profile conflicts with the proposed names;
- that a rollback archive exists before each configuration stage.

## Phase 1: Anton prepares Cloudflare

These steps require Anton's Cloudflare account and domain authority.

1. Sign in to Cloudflare.
2. Ensure the chosen domain is active on Cloudflare DNS.
3. Go to **Networking → Tunnels** and create a remotely managed tunnel, for example `hermes-chocobot`.
4. Choose Linux as the connector platform.
5. Do not paste the connector token into chat or commit it. Either:
   - run Cloudflare's one-time installation command directly on `hermes-box`; or
   - provide ChocoBot access to a secure local credential/command handoff approved for this installation.
6. Add a published application route:
   - hostname: the chosen ChocoBot webhook hostname;
   - service: `http://127.0.0.1:8644`.
7. Confirm the tunnel appears in the Cloudflare dashboard. It may remain unhealthy until the connector is installed and running.

Do not add a Cloudflare Access login policy in front of the webhook path: GitHub webhook deliveries cannot complete an interactive login. Authentication at this endpoint is GitHub's HMAC signature. Other hostnames or paths may use Access independently.

### ChocoBot completes the host side

After Anton authorizes the tunnel, ChocoBot can:

1. create a fresh Hermes backup;
2. install `cloudflared` through Cloudflare's supported Linux package/service workflow;
3. install/start the durable connector service;
4. verify it starts after reboot;
5. keep the origin bound to loopback and avoid router port forwarding;
6. verify the public hostname has valid TLS and reaches the local health endpoint after Phase 2.

If installation requires `sudo`, Anton must approve the exact command and may need to enter credentials locally.

## Phase 2: ChocoBot prepares Hermes

Anton does not need to edit Hermes files manually. After confirming the scope, ChocoBot can perform these steps through official Hermes commands and supported configuration paths.

1. Run a labeled quick backup:

   ```bash
   hermes backup --quick --label before-chocobot-reviews
   ```

2. Create a dedicated profile:

   ```bash
   hermes profile create prreviewer
   ```

3. Configure/authenticate its model provider without cloning Anton's entire default profile.
4. Keep approvals in `smart` mode.
5. Configure the Docker terminal backend and minimal webhook tool surface.
6. Disable Home Assistant, messaging, memory, cron, browser/media, and unrelated MCP access for webhook runs.
7. Enable gateway profile multiplexing.
8. Enable the Hermes webhook adapter on `127.0.0.1:8644` with a strong secret.
9. Restart the gateway and verify:

   ```bash
   curl http://127.0.0.1:8644/health
   hermes gateway status
   ```

10. Create and fixture-test the repository/action filter.
11. Create the `duplex-voice-pr-review` route with the `github-code-review` skill and log-only response delivery. The review agent itself publishes the single formal GitHub review.
12. Verify the public health endpoint through Cloudflare.

The final webhook URL is:

```text
https://<chosen-hostname>/p/prreviewer/webhooks/duplex-voice-pr-review
```

ChocoBot must stop if the installed Hermes version cannot keep the GitHub App private key outside the review agent's capability surface.

## Phase 3: Anton creates the ChocoBot GitHub App

Anton performs this phase because GitHub requires account-owner interaction.

1. Open **GitHub → Settings → Developer settings → GitHub Apps → New GitHub App**.
2. Follow [Spec 014](../.agents/specs/014-create-chocobot-github-app.md) exactly.
3. Use `ChocoBot` as the name if available. GitHub App names are globally unique; report the final app slug if a variant is required.
4. Use the repository URL as the homepage.
5. Leave callback URL, OAuth installation authorization, Device Flow, and setup URL disabled/blank.
6. Enable the webhook with:
   - the final profile-qualified HTTPS URL from Phase 2;
   - the secret generated on `hermes-box` through the agreed secure handoff;
   - SSL verification enabled.
7. Grant only:
   - Contents: read-only;
   - Pull requests: read/write;
   - Checks: read-only;
   - Commit statuses: read-only;
   - Metadata: read-only/platform-required.
8. Subscribe only to **Pull request**.
9. Choose **Only on this account**.
10. Create the app.
11. Install it for `anton-matosov`, choose **Only select repositories**, and select only `hermes-duplex-voice`.

Do not grant Administration, workflow write, actions write, deployments, secrets, organization permissions, or broad repository access.

## Phase 4: Anton hands off app credentials safely

From the GitHub App settings page:

1. Record the numeric **App ID**. The App ID is not the client ID.
2. Generate a private key and download the `.pem` file.
3. Move the key to the secure path agreed with ChocoBot, outside this repository.
4. Tell ChocoBot only:
   - App ID;
   - installed app slug/bot login;
   - private-key **path**, not its contents;
   - installation completed for the target repository.
5. Do not paste the PEM or webhook secret into chat.

ChocoBot then:

1. verifies key ownership and owner-only permissions;
2. verifies the key is not under the repository, shared attachment, or review-container mounts;
3. resolves and records the installation ID through GitHub's API;
4. implements/tests the constrained installation-token broker;
5. confirms the installation token sees exactly the intended repository and permissions;
6. confirms forbidden operations fail.

Installation tokens expire after one hour. ChocoBot mints them on demand; Anton should never maintain a permanent ChocoBot PAT.

## Phase 5: Joint validation

### Safe tests ChocoBot can run without posting a review

- local filter fixtures for every accepted/rejected action;
- valid/invalid webhook HMAC requests;
- malformed and oversized payload handling;
- Cloudflare and Hermes health checks;
- profile isolation and private-key non-readability;
- GitHub App installation-token creation and read-only API calls;
- permission-denied tests for prohibited operations;
- deduplication logic using fixtures.

### The test Anton must authorize

A real review is an external, persistent GitHub action. Anton chooses a disposable PR or nominates an open PR and explicitly authorizes one live smoke review.

ChocoBot then verifies:

1. GitHub delivered the signed event successfully.
2. Only the restricted `prreviewer` profile ran.
3. The reviewed head SHA matches the PR head immediately before publication.
4. One formal `COMMENT` review appears from `chocobot[bot]` or the actual app bot slug.
5. Inline comments, if any, attach to changed lines.
6. The review marker contains the reviewed head SHA.
7. ChocoBot reads the review and comments back successfully.
8. Replaying another accepted action for the same head does not post a duplicate.

## What normal operation looks like

After setup, Anton only needs to open, reopen, or assign a PR. No manual `@mention`, workflow dispatch, or reviewer request is required.

For each accepted event, ChocoBot:

1. authenticates the webhook;
2. filters repository/action/assignee;
3. pins the PR head;
4. skips if that head already has a ChocoBot marker;
5. reviews the complete diff and runs available checks;
6. rechecks the head;
7. posts one advisory `COMMENT` review;
8. reads it back and records the review URL;
9. cleans up the temporary checkout.

ChocoBot does not approve, request changes, merge, push fixes, or alter branches.

## Monitoring and troubleshooting

ChocoBot can inspect:

```bash
hermes gateway status
hermes webhook list
systemctl status cloudflared
```

Also inspect:

- GitHub App **Advanced → Recent deliveries** for status, request headers, and redelivery;
- Hermes gateway logs for route decisions and agent failures;
- Cloudflare tunnel health for connector failures;
- ChocoBot review marker and head SHA for deduplication.

Common failure boundaries:

| Symptom | Likely cause |
|---|---|
| GitHub cannot deliver | tunnel/DNS/service unavailable |
| HTTP 401 | webhook secret mismatch or missing signature |
| Delivery succeeds but no run | event/action/repository/assignee filtered out |
| GitHub API 401 | expired/invalid installation token or JWT |
| GitHub API 403 | app permission or installation scope insufficient |
| Review author is Anton | personal token used instead of app installation token |
| Duplicate reviews | missing marker/head deduplication |
| Inline comment rejected | stale head or line not present in current diff |
| Review runtime sees host files | Docker/profile isolation misconfigured; stop automation |

## Pausing and rollback

To pause safely:

1. Disable the GitHub App webhook or suspend/uninstall the app.
2. Remove or disable the Hermes subscription:

   ```bash
   hermes webhook remove duplex-voice-pr-review
   ```

3. Stop the tunnel if it has no other routes.

For full rollback, ChocoBot can restore the labeled Hermes backup, remove the `prreviewer` profile after checking it has no other users, remove the filter/broker, and remove the tunnel service/hostname. Anton removes the GitHub App installation and deletes app private keys in GitHub. Verify with a fresh API call that installation access is gone.

## Credential rotation

Anton generates a replacement GitHub App private key before revoking the old one. ChocoBot installs and verifies the new key, confirms token minting and a read-only API call, then Anton deletes the old key. Rotate the webhook secret separately with an ordered GitHub/Hermes update and a signed delivery test.

Never paste keys or secrets into an issue, PR, chat, shell history, or committed file.

## Handoff checklist

Anton can provide the following non-secret values when ready:

```text
Cloudflare domain:
Chosen webhook hostname:
Tunnel name:
Final GitHub App name/slug:
App ID:
Private-key path on hermes-box:
App installed only on anton-matosov/hermes-duplex-voice: yes/no
Review drafts when opened: yes/no
Add synchronize later: yes/no
PR authorized for live smoke review:
```

Do not include the PEM contents, webhook secret, connector token, JWT, or installation token in this handoff.

## References

- [Hermes webhooks](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- [Cloudflare: create a remotely managed tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/)
- [GitHub: register a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)
- [GitHub: install your own GitHub App](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app)
