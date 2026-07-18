# Spec 014: Create the ChocoBot GitHub App

## Objective

Create a private GitHub App named `ChocoBot` that receives pull-request events for `anton-matosov/hermes-duplex-voice` and publishes least-privilege automated code reviews as a distinct bot identity.

The app is an identity, permission boundary, webhook source, and short-lived credential issuer. It is not an OAuth login app, does not act as Anton, and must not receive authority to merge, push code, manage repository settings, read secrets, or dispatch workflows.

## Dependencies

- A stable HTTPS webhook URL supplied by [Spec 013](013-automated-chocobot-code-reviews.md):

  ```text
  https://<chosen-hostname>/p/prreviewer/webhooks/duplex-voice-pr-review
  ```

- Anton's GitHub account must own `anton-matosov/hermes-duplex-voice` and be allowed to register/install a GitHub App.
- A secure host path for the generated private key, outside the repository and outside the review agent's file/terminal roots.

GitHub App names are globally unique. Try `ChocoBot`; if GitHub reports that name is unavailable, choose the smallest recognizable variant and record the resulting app slug and bot login. Do not silently substitute a personal access token.

## Ownership split

### Anton performs in GitHub's UI

- register the app under the intended owner;
- enter and confirm URLs, permissions, event subscription, and installation scope;
- generate/download the private key;
- install the app on only the target repository;
- approve any future permission expansion.

### ChocoBot prepares and verifies

- generate the webhook secret locally and give Anton a secure way to place it in GitHub without committing or pasting it into chat;
- provide the exact field values below;
- secure and validate the private-key file after Anton places it on the host;
- discover and record the app ID and installation ID without exposing credentials;
- implement short-lived installation-token minting and constrained API access;
- run read-only permission tests before the first explicitly authorized review-write test.

## Registration settings

Navigate to **GitHub → Settings → Developer settings → GitHub Apps → New GitHub App** and use:

| Field | Required value |
|---|---|
| GitHub App name | `ChocoBot` if available |
| Description | `Automated, evidence-based code reviews by ChocoBot for Hermes Duplex Voice.` |
| Homepage URL | `https://github.com/anton-matosov/hermes-duplex-voice` |
| Callback URL | blank |
| Request user authorization during installation | disabled |
| Device Flow | disabled |
| Setup URL | blank |
| Webhook | active |
| Webhook URL | stable Spec 013 profile-qualified HTTPS URL |
| Webhook secret | strong locally generated secret shared with the Hermes route |
| SSL verification | enabled |
| Installation scope | **Only on this account** |

Do not enable OAuth/user authorization. ChocoBot acts as an app installation, not on behalf of a user.

## Repository permissions

Start with the following minimum permissions:

| Permission | Level | Reason |
|---|---:|---|
| Metadata | Read-only | Repository identity and basic metadata; GitHub Apps receive metadata access as required by the platform |
| Contents | Read-only | Read source, refs, and commits; authenticate an HTTPS clone if needed |
| Pull requests | Read & write | Read PR metadata/diffs/reviews and create a formal PR review |
| Checks | Read-only | Inspect check runs without changing them |
| Commit statuses | Read-only | Inspect legacy status contexts without changing them |

Set every other repository permission to **No access**, including:

- Administration;
- Actions write access;
- Workflows;
- Environments and deployments;
- Issues, unless a later feature explicitly requires issue comments;
- Secrets and variables;
- Webhooks;
- Members, organization administration, and all organization permissions.

If GitHub's current registration UI uses different permission labels, map them to the documented REST endpoint requirements and choose the least privilege that satisfies:

- read pull request metadata, files, diff, reviews, and review comments;
- read checks/statuses;
- `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews`;
- read the created review and comments back.

A `403` must trigger permission diagnosis from `X-Accepted-GitHub-Permissions`; it is not justification to grant broad administration or classic `repo` scope.

## Event subscription

Subscribe only to:

- **Pull request** (`pull_request`).

GitHub App event subscriptions select an event family, not individual actions. The Hermes route enforces the `opened`, `reopened`, and Anton-specific `assigned` predicate.

Do not subscribe to push, pull-request review, issue comment, workflow, check, deployment, installation, organization, membership, or repository-administration events unless a later spec requires them.

## Installation

After creating the app:

1. Open the app's **Install App** page.
2. Install it for `anton-matosov`.
3. Choose **Only select repositories**.
4. Select only `hermes-duplex-voice`.
5. Confirm the requested permissions match this spec before installing.
6. Record the installation ID from the installation URL or GitHub API.
7. Verify the installation repository list contains exactly `anton-matosov/hermes-duplex-voice`.

The app must not be installable by arbitrary accounts for the initial release.

## Private key and secret handling

### Webhook secret

- Generate at least 32 random bytes using a cryptographically secure generator.
- Store it through Hermes's supported secret/configuration flow.
- Enter the same value in the GitHub App webhook settings.
- Never commit it, include it in docs/fixtures, print it in logs, or paste it into chat.
- Rotating it requires an ordered update so GitHub and Hermes continue to agree; verify a signed delivery after rotation.

### App private key

1. Generate a private key from the GitHub App settings page.
2. Download the `.pem` once.
3. Move it to the pre-agreed host credential directory outside the repository.
4. Set owner-only permissions and verify ownership.
5. Ensure it is excluded from backups that leave the trusted host, crash reports, agent attachments, shell history, and repository searches.
6. Do not mount it into the review container or make it readable through Hermes file/terminal tools.
7. Generate a replacement key before deleting the old key during rotation; verify token minting with the replacement, then revoke the old key.

The private key is long-lived and high impact. Installation tokens are short-lived and expire after one hour; the broker must mint them on demand rather than caching them as permanent `GITHUB_TOKEN` values.

## Credential broker contract

Implement a host-side broker or narrowly scoped Hermes integration that:

1. Signs a GitHub App JWT using the app ID and private key without returning the private key.
2. Uses the installation ID to request an installation access token.
3. Optionally narrows the token request to the target repository and permissions.
4. Keeps the token out of model context, command lines, logs, and persistent session history.
5. Exposes only named read/review operations needed by Spec 013.
6. Rejects any owner/repository other than `anton-matosov/hermes-duplex-voice`.
7. Rejects merge, push, branch mutation, administration, workflow dispatch, secrets, release, issue mutation, and arbitrary URL/method access.
8. Refreshes an expired token and retries a failed authenticated request at most once.
9. Redacts new stateless installation-token formats as opaque secrets; never assume tokens have a fixed length.
10. Supports key rotation without changing the agent prompt or repository.

## Verification

### Registration checks

- App owner, app ID, slug, bot login, homepage, and private visibility match expectations.
- OAuth callback/user authorization and Device Flow are disabled.
- Webhook is active, URL is exact, SSL verification is enabled, and a secret is configured.
- The only subscribed event is `pull_request`.
- Permissions match the least-privilege table.
- Installation scope is only the owning account.
- Installation includes exactly the target repository.

### Authentication checks

- Generate a valid app JWT without logging it.
- Resolve the installation ID independently through GitHub's API.
- Mint an installation token and confirm its expiry is no more than GitHub's installation-token lifetime.
- Confirm repository scope contains only the target repository.
- Confirm read calls for PR metadata, files/diff, reviews, comments, checks, and statuses succeed.
- Confirm a disallowed operation such as repository administration or workflow dispatch is rejected by permissions and by the local broker.
- Confirm the private key is unreadable from the review container and agent tools.
- Confirm token and key patterns are redacted from logs.

### Webhook checks

- GitHub's delivery page shows the expected endpoint and `pull_request` event.
- Hermes accepts a valid `X-Hub-Signature-256` signature and rejects an invalid one.
- GitHub's ping does not start a review because the Hermes route accepts only `pull_request`.
- A non-matching action is ignored.

### Review-write smoke

Only after Anton explicitly authorizes a real external write:

1. Use a disposable PR or nominated open PR.
2. Create one formal review with event `COMMENT` and a unique test marker.
3. Read the review back.
4. Verify the author is the app bot identity, commit ID is correct, and no personal Anton credential was used.
5. Remove no review history unless Anton explicitly requests cleanup; GitHub reviews are audit records.

## Rotation and revocation

- Review app permissions and installation repository scope periodically.
- Rotate the private key on a documented schedule and immediately after suspected exposure.
- Rotate the webhook secret independently when needed.
- Suspend or uninstall the app to stop API access; disable the Hermes subscription first to avoid repeated failed runs.
- Delete all private keys from GitHub and the host when retiring the app.
- Verify a fresh API query after suspension/uninstall confirms access is gone.

## Acceptance criteria

1. GitHub displays automated reviews as the ChocoBot app bot, not `anton-matosov`.
2. The app is private to its owner and installed only on `anton-matosov/hermes-duplex-voice`.
3. Only `pull_request` webhooks are subscribed.
4. Permissions are limited to contents read, pull requests read/write, and check/status reads needed for review.
5. OAuth user authorization is disabled.
6. Webhook SSL verification and HMAC secret validation are enabled.
7. The app private key never reaches the review model/container.
8. Installation tokens are minted on demand, expire, remain secret, and are constrained by a broker.
9. Allowed review reads/writes succeed; prohibited repository operations fail.
10. A real smoke review is posted only with Anton's explicit authorization and is read back successfully.

## Non-goals

- a public GitHub Marketplace app;
- installation on other repositories or accounts;
- OAuth login or acting on behalf of Anton;
- a personal access token fallback;
- issue triage, code modification, branch pushes, merges, releases, deployments, or workflow dispatch;
- repository or organization administration;
- check-run creation or required-status enforcement;
- changing the Spec 013 trigger policy.

## References

- [Registering a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)
- [Choosing permissions for a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)
- [Installing your own GitHub App](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app)
- [Generating an installation access token](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app)
- [Pull-request review REST endpoints](https://docs.github.com/en/rest/pulls/reviews#create-a-review-for-a-pull-request)
