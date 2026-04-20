---
name: xhost
description: >-
  Use when the user wants to deploy a website, host a static site, put something
  online, publish a page, create a preview URL, check deployment status, or
  mentions xhost. Handles account setup, app creation, git push, deploy status,
  and preview channels via the xhost hosting platform.
tools: Bash, Read, Glob, Grep
---

# xhost Static Site Hosting

xhost is a hosting platform for static sites and dynamic applications. You push code to a git remote and then trigger a deploy via the API. Every app gets a production URL, and you can create preview URLs for branches.

## Prerequisites

Two environment variables control xhost access:

- **XHOST_TOKEN** — Your API token (starts with `xh_`). Required for all authenticated operations.
- **XHOST_API_URL** — The API base URL. Defaults to `https://api.xhostd.com`.

Check if they are set:
```bash
[ -n "$XHOST_TOKEN" ] && echo "Token is set" || echo "Token not set"
echo "API URL: ${XHOST_API_URL:-https://api.xhostd.com}"
```

If the token is not set, guide the user through signup or login. See `/xhost:login` or `/xhost:signup` for explicit flows.

## Account Setup

New users need an **invite code** (format: `xi_...`) from an admin. Signup is a single API call:

```bash
curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/admin/signup" \
  -H "Content-Type: application/json" \
  -d '{"invite":"xi_...","username":"chosen-name","email":"user@example.com"}'
```

The response includes a `token` field (`xh_...`) that the user must export as `XHOST_TOKEN`.

Usernames must follow DNS label rules: lowercase letters, digits, hyphens, 1-40 characters, no leading or trailing hyphen.

## Creating Apps

Each app is a git repository with at least one channel (the `prod` channel, created automatically).

```bash
curl -sf -X POST "${XHOST_API_URL}/apps" \
  -H "Authorization: Bearer $XHOST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-app"}'
```

The response includes `repo_url` (the git remote to push to) and a `channels` array with the prod channel's `hostname`. App names follow the same DNS label rules as usernames, plus they cannot use reserved prefixes: `git`, `api`, `www`, `admin`, `preview`, `staging`.

After creating the app, add the git remote with the token embedded for push authentication:
```bash
git remote add xhost "https://${XHOST_TOKEN}@git.xhostd.com/<username>/<app>.git"
```

## How Deploys Work

Deploying is a two-step process: push code, then trigger the deploy.

```bash
# 1. Push code to the xhost remote
git push xhost master

# 2. Trigger the deploy
SHA=$(git rev-parse HEAD)
curl -sf -X POST "${XHOST_API_URL}/apps/<app_id>/channels/<channel_id>/deploy" \
  -H "Authorization: Bearer $XHOST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sha":"'"$SHA"'"}'
```

The deploy pipeline clones the repo, builds a container, and routes traffic to it. Use the `/xhost:deploy` skill to automate both steps.

The production channel is bound to `branch:master` by default.

## Preview Branches

To deploy a preview of a feature branch:

1. Ensure a `branch:*` wildcard channel exists on the app. If not, create one:
   ```bash
   curl -sf -X POST "${XHOST_API_URL}/apps/<app_id>/channels" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"wildcard","git_ref_binding":"branch:*"}'
   ```

2. Push the branch:
   ```bash
   git push xhost feature-branch
   ```

3. Trigger a deploy for the branch (the fan-out creates a `preview-<slug>` child channel automatically when the deploy API resolves the wildcard binding):
   ```bash
   SHA=$(git rev-parse HEAD)
   curl -sf -X POST "${XHOST_API_URL}/apps/<app_id>/channels/<channel_id>/deploy" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"sha":"'"$SHA"'"}'
   ```
   List channels after a few seconds to find the preview URL.

Preview channels are cleaned up automatically when the branch is deleted from git (via the sweeper).

## Checking Status

```bash
curl -sf "${XHOST_API_URL}/apps" -H "Authorization: Bearer $XHOST_TOKEN"
```

This returns all apps with their channels. Each channel includes:
- `hostname` — the URL where the channel is served (present as `https://<hostname>`)
- `git_ref_binding` — which branch is bound for deploys (e.g., `branch:master`, `branch:*`, `branch:feature-x`)
- `current_sha` — the currently deployed commit (first 7 chars), or null if not yet deployed
- `status` — one of: `provisioning`, `running`

## Key API Details

- **Base URL**: `${XHOST_API_URL:-https://api.xhostd.com}`
- **Auth header**: `Authorization: Bearer $XHOST_TOKEN`
- **Error envelope**: `{"error": {"code": "...", "message": "..."}}` — always show the `message` to the user
- **Token format**: `xh_` prefix
- **Invite format**: `xi_` prefix

Common error codes:
- `token_invalid` — bad or missing token
- `bad_request` — validation failure (message has details)
- `not_found` — app or channel does not exist
- `scope_denied` — token lacks required permission

## Slash Command

For explicit step-by-step workflows, users can invoke `/xhost:login`, `/xhost:signup`, `/xhost:init`, `/xhost:deploy`, `/xhost:preview`, `/xhost:status`.

## References

For complete API details, see the reference files in this skill:
- `references/api-reference.md` — Full endpoint documentation
- `references/getting-started.md` — Step-by-step new user guide
