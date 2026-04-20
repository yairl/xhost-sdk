---
name: deploy
description: >-
  Use when the user wants to deploy, push to production, or ship their site/app to xhost.
tools: Bash, Read, Glob, Grep
---

# xhost — Deploy

## Current Context

- Working directory: !`pwd`
- Git remotes: !`git remote -v`
- Current branch: !`git branch --show-current`
- Latest commit: !`git log --oneline -1`

## Pre-flight

Before starting, run `printenv XHOST_TOKEN` to verify the user has a token set. If not, direct them to `/login` or `/signup`.

## Deploy to production

Commit, push, and trigger a deploy on xhost.

1. **Check XHOST_TOKEN** — if not set, tell the user to run `/xhost:login` or `/xhost:signup` first and stop.
2. **Check xhost remote** — run `git remote get-url xhost`. If it does not exist, suggest `/xhost:init` and stop.
3. **Check for uncommitted changes** — run `git status --short`. If there are changes:
   - Stage all changes: `git add -A`
   - Create a commit with a descriptive message summarizing the changes.
4. **Push** code to the xhost remote:
   ```
   git push xhost master
   ```
   (Or substitute the current branch name if not on master.)
5. **Get the pushed SHA**:
   ```
   SHA=$(git rev-parse HEAD)
   ```
6. **Find the app and channel** — list apps to find the matching app and its prod channel:
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps" -H "Authorization: Bearer $XHOST_TOKEN"
   ```
   Match the app whose `repo_url` corresponds to the xhost remote URL (strip the token from the remote URL before comparing). Find the `prod` channel's `id` and `hostname`.
7. **Trigger the deploy**:
   ```
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels/<channel_id>/deploy" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"sha":"<SHA>"}'
   ```
8. **Check deploy status** — the deploy response returns `{"deploy_id": "...", "status": "queued"}`. Poll or check the deploy log:
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels/<channel_id>/logs?deploy=<deploy_id>" \
     -H "Authorization: Bearer $XHOST_TOKEN"
   ```
9. Report: **"Deployed! Live at https://<prod_hostname>"**

**Important:** pushing code and deploying are separate steps. `git push` stores code on the server; the deploy API call triggers the build and start.

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
