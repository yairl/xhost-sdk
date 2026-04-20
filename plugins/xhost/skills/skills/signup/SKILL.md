---
name: signup
description: >-
  Use when the user wants to create a new xhost account or sign up for xhost.
tools: Bash, Read, Glob, Grep
---

# xhost — Signup

## Pre-flight

Before starting, run `printenv XHOST_TOKEN` to check if the user already has a token set. If they do, let them know and ask if they still want to create a new account.

## Create a new xhost account

Create a new account without leaving Claude Code.

1. Ask the user for their **invite code** (format: `xi_...`). Every new account requires one.
2. Ask for a **username**. Rules:
   - Lowercase letters, digits, and hyphens only
   - 1-40 characters
   - No leading or trailing hyphen
   - Regex: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`
3. Optionally ask for an **email** address.
4. Call the signup endpoint:
   ```
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/admin/signup" \
     -H "Content-Type: application/json" \
     -d '{"invite":"<invite_code>","username":"<username>","email":"<email_or_null>"}'
   ```
5. On success: the response contains `{"user_id":"...","username":"...","email":"...","token":"xh_..."}`. Extract the `token` field and tell the user:
   - `export XHOST_TOKEN=xh_...` (session)
   - Add to shell profile for persistence
6. Common errors:
   - `"invite is invalid or already used"` — the invite code is wrong or was already consumed
   - `"username is already taken"` — pick a different username
   - Validation errors if the username does not match DNS label rules

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
