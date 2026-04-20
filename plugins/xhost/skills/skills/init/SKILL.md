---
name: init
description: >-
  Use when the user wants to create an xhost app and connect this project to xhost for hosting. Supports static sites, Node.js, and Python applications.
tools: Bash, Read, Glob, Grep, Write
---

# xhost — Init

## Current Context

- Working directory: !`pwd`
- Git remotes: !`git remote -v`
- Current branch: !`git branch --show-current`
- Latest commit: !`git log --oneline -1`

## Pre-flight

Before starting, run `printenv XHOST_TOKEN` to verify the user has a token set. If not, direct them to `/login` or `/signup`.

## Create an app and connect this project

Set up a new xhost app linked to the current git repository.

1. **Check XHOST_TOKEN** — if not set, tell the user to run `/xhost:login` or `/xhost:signup` first and stop.
2. **Check XHOST_API_URL** — if not set, default to `https://api.xhostd.com`. Mention this to the user so they know which instance they are using.
3. **Derive app name** from the current directory name. Slugify it to DNS label rules (lowercase, replace non-alphanumeric with hyphens, trim leading/trailing hyphens, max 40 chars). App names cannot start with these reserved prefixes: `git`, `api`, `www`, `admin`, `preview`, `staging` (exact match or followed by `-`). Ask the user to confirm the app name.
4. **Detect template and generate scripts** — follow this detection order:
   a. **launch.sh exists** — if `launch.sh` already exists in the working directory, use `template: "app"` and do not overwrite any scripts. Tell the user: "Found existing launch.sh. Using app template."
   b. **Node.js project** — if `package.json` exists with a `scripts.start` field:
      - Generate `install.sh`:
        ```sh
        #!/bin/sh
        set -e
        if [ -f package-lock.json ]; then npm ci; else npm install; fi
        ```
      - Generate `launch.sh`:
        ```sh
        #!/bin/sh
        set -e
        if node -e "process.exit(require('./package.json').scripts?.build?0:1)" 2>/dev/null; then
            npm run build
        fi
        exec npm start
        ```
      - Tell the user: "Detected Node.js project. Created install.sh and launch.sh. xhost will install dependencies and start your app via `npm start`."
      - Template: `"app"`.
   c. **Python project** — if `requirements.txt` exists:
      - Generate `install.sh`:
        ```sh
        #!/bin/sh
        set -e
        uv pip install --system --no-cache -r requirements.txt
        ```
      - Detect the entry point by checking for common files in order: `app.py`, `main.py`, `manage.py`. Use the first one found. If none exist, default to `app.py` and tell the user they may need to edit `launch.sh`.
      - Generate `launch.sh`:
        ```sh
        #!/bin/sh
        set -e
        exec python <entry_point>
        ```
      - Tell the user: "Detected Python project. Created install.sh and launch.sh."
      - Template: `"app"`.
   d. **Otherwise** — template `"static"`. No scripts needed.
5. **Create the app**:
   ```
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"<app_name>","template":"<template>"}'
   ```
6. Parse the response. It returns:
   ```json
   {
     "id": "uuid",
     "name": "app-name",
     "repo_url": "https://git.<domain>/<username>/<app>.git",
     "template": "static",
     "channels": [{"name": "prod", "hostname": "<app>-<user>.<domain>", ...}]
   }
   ```
7. **Construct the authenticated git remote URL**: take `repo_url` and insert the token before the hostname. For example, if `repo_url` is `https://git.xhostd.com/alice/myapp.git`, the remote URL becomes `https://xh_abc123@git.xhostd.com/alice/myapp.git`.
8. **Add the git remote**:
   ```
   git remote add xhost <url_with_token>
   ```
9. If there are no commits in the repo, stage everything and create an initial commit:
   ```
   git add . && git commit -m "initial commit"
   ```
   If the generated `install.sh` and `launch.sh` are not yet committed, stage and commit them:
   ```
   git add install.sh launch.sh && git commit -m "add xhost deploy scripts"
   ```
10. **Push**:
    ```
    git push xhost master
    ```
11. **Deploy** — get the SHA and trigger the deploy:
    ```
    SHA=$(git rev-parse HEAD)
    ```
    Find the app's prod channel ID from the create response (step 6), then:
    ```
    curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels/<channel_id>/deploy" \
      -H "Authorization: Bearer $XHOST_TOKEN" \
      -H "Content-Type: application/json" \
      -d '{"sha":"'"$SHA"'"}'
    ```
12. Report success:
    - "App created and deployed! Live at https://<hostname>"
    - Mention that future deploys are two steps: `git push xhost master` followed by `/xhost:deploy` (or an explicit deploy API call).
    - For app-template projects: mention that the first deploy takes 30-90 seconds (install.sh runs on each deploy). Subsequent deploys that don't change dependencies will be faster once caching ships.

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
