---
name: login
description: >-
  Use when the user wants to configure their xhost token or log in to xhost.
tools: Bash, Read, Glob, Grep
---

# xhost — Login

## Pre-flight

Before starting, run `printenv XHOST_TOKEN` to check if the user already has a token set.

## Configure an existing token

The user already has an xhost account and token. Help them configure it.

1. Ask the user for their `xh_*` token.
2. Validate the token by calling:
   ```
   curl -sf -H "Authorization: Bearer <token>" ${XHOST_API_URL:-https://api.xhostd.com}/apps
   ```
3. If the request fails, tell the user the token is invalid or the API is unreachable.
4. If valid, tell the user to set it:
   - For the current session: `export XHOST_TOKEN=xh_...`
   - For persistence: add `export XHOST_TOKEN=xh_...` to their shell profile (`~/.bashrc`, `~/.zshrc`, etc.)
5. Report success. If the response contains app data, mention the username or number of apps.

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
