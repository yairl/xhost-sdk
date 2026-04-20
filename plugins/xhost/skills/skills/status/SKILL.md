---
name: status
description: >-
  Use when the user wants to check deployment status, see their apps, channels, or hostnames on xhost.
tools: Bash, Read, Glob, Grep
---

# xhost — Status

## Current Context

- Working directory: !`pwd`
- Git remotes: !`git remote -v`
- Current branch: !`git branch --show-current`
- Latest commit: !`git log --oneline -1`

## Pre-flight

Before starting, run `printenv XHOST_TOKEN` to verify the user has a token set. If not, direct them to `/login` or `/signup`.

## Show deployment status

Display apps, channels, hostnames, and current deploy state.

1. **Check XHOST_TOKEN** — if not set, tell the user to run `/xhost:login` or `/xhost:signup` first and stop.
2. **List apps**:
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps" -H "Authorization: Bearer $XHOST_TOKEN"
   ```
3. If the `xhost` git remote exists, show only the app whose `repo_url` matches. Otherwise, show all apps.
4. For each app, display a table or formatted list:
   - **App name**
   - For each channel:
     - Channel name
     - URL: `https://<hostname>`
     - Binding: `git_ref_binding` value
     - SHA: first 7 characters of `current_sha`, or "not deployed"
     - Status: `status` value

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
