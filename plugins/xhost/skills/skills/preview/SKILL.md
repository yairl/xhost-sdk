---
name: preview
description: >-
  Use when the user wants to deploy a preview of their current branch, create a preview URL, or test a branch on xhost.
tools: Bash, Read, Glob, Grep
---

# xhost — Preview

## Current Context

- Working directory: !`pwd`
- Git remotes: !`git remote -v`
- Current branch: !`git branch --show-current`
- Latest commit: !`git log --oneline -1`

## Pre-flight

Before starting, run `printenv XHOST_TOKEN` to verify the user has a token set. If not, direct them to `/login` or `/signup`.

## Deploy a preview of the current branch

Push the current branch and get a preview URL.

1. **Check XHOST_TOKEN** — if not set, tell the user to run `/xhost:login` or `/xhost:signup` first and stop.
2. **Check xhost remote** — if missing, suggest `/xhost:init` and stop.
3. **Get current branch**. If on `master`, warn the user: "You are on the master branch, which deploys to production. Consider creating a feature branch first (e.g. `git checkout -b my-feature`)." Ask if they want to continue or create a branch.
4. **Find the app** — list apps via `GET /apps`, match against the xhost remote's `repo_url`.
5. **Check for a `branch:*` wildcard channel**:
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels" \
     -H "Authorization: Bearer $XHOST_TOKEN"
   ```
   Look for a channel with `"git_ref_binding": "branch:*"`.
6. If no wildcard channel exists, **create one**:
   ```
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"wildcard","git_ref_binding":"branch:*"}'
   ```
7. **Commit any uncommitted changes** (same as deploy: stage all, commit with descriptive message).
8. **Push the current branch**:
   ```
   git push xhost <branch_name>
   ```
9. **Trigger the deploy** — get the SHA and call the deploy API on the prod channel (the wildcard binding will fan out to create a preview child channel automatically):
   ```
   SHA=$(git rev-parse HEAD)
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels/<prod_channel_id>/deploy" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"sha":"'"$SHA"'"}'
   ```
   Alternatively, call `POST /internal/deploy` with `{"user_id": "...", "repo_name": "...", "ref": "refs/heads/<branch_name>", "sha": "<SHA>"}` if using the internal API.
10. **Wait briefly** (~5 seconds), then list channels to find the new preview channel:
    ```
    curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels" \
      -H "Authorization: Bearer $XHOST_TOKEN"
    ```
    Look for a channel whose `git_ref_binding` is `branch:<branch_name>` and whose name starts with `preview-`.
11. Report: **"Preview deployed! URL: https://<preview_hostname>"**

---

## Common Patterns

Use these patterns across all operations:

- **API base URL**: `${XHOST_API_URL:-https://api.xhostd.com}`. If `XHOST_API_URL` is not set, tell the user to export it before proceeding (unless using the default).
- **Auth header**: `-H "Authorization: Bearer $XHOST_TOKEN"`
- **Error envelope**: The API returns errors as `{"error": {"code": "...", "message": "..."}}`. Always show the `message` field to the user. Never show raw JSON or stack traces.
- **App resolution**: To find which app matches the current project, compare the `xhost` git remote URL (with the token stripped out) against `repo_url` values from `GET /apps`.
- **Hostnames**: Always present URLs as `https://<hostname>`.
- **Two-step deploy**: Push stores code; `POST /apps/{id}/channels/{id}/deploy` triggers the actual build and deploy. Always do both steps.
- **DNS label rules**: Lowercase letters, digits, hyphens. 1-40 characters. No leading/trailing hyphen. Regex: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`
- **Reserved app name prefixes**: `git`, `api`, `www`, `admin`, `preview`, `staging` (blocked as exact match or prefix followed by `-`).
