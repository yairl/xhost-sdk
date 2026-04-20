# Getting Started with xhost

This guide walks you through setting up your first static site on xhost, from account creation to live deployment.

## 1. Get an Invite Code

Every new xhost account requires an invite code (format: `xi_...`). Ask your team admin or the xhost operator to generate one for you. Admins create invites via the API:

```bash
curl -sf -X POST "${XHOST_API_URL}/admin/invites" \
  -H "Authorization: Bearer <admin_token>"
```

## 2. Sign Up

You can sign up in two ways:

**Option A: Using the `/xhost:signup` command in Claude Code**

Type `/xhost:signup` and follow the prompts. You will be asked for your invite code, a username, and optionally an email.

**Option B: Using curl directly**

```bash
curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/admin/signup" \
  -H "Content-Type: application/json" \
  -d '{"invite":"xi_your_code","username":"your-username","email":"you@example.com"}'
```

Username rules:
- Lowercase letters, digits, and hyphens only
- 1 to 40 characters
- Cannot start or end with a hyphen

On success, the response includes your API token:
```json
{
  "user_id": "...",
  "username": "your-username",
  "token": "xh_your_token_here"
}
```

## 3. Set Your Token

Save the token so xhost commands can authenticate:

**For the current session:**
```bash
export XHOST_TOKEN=xh_your_token_here
```

**For persistence** (add to your shell profile):
```bash
echo 'export XHOST_TOKEN=xh_your_token_here' >> ~/.bashrc
# or ~/.zshrc, depending on your shell
```

Also set the API URL if you are not using the default (`https://api.xhostd.com`):
```bash
export XHOST_API_URL=https://api.xhostd.com
```

## 4. Create Your First App

Navigate to your project directory (a git repository with your static site) and run:

```
/xhost:init
```

This will:
1. Derive an app name from your directory name
2. Create the app on xhost (which provisions a git repo and a production channel)
3. Add an `xhost` git remote to your local repo
4. Push your code to trigger the first deploy

You can also do this manually:

```bash
# Create the app
curl -sf -X POST "${XHOST_API_URL}/apps" \
  -H "Authorization: Bearer $XHOST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-site"}'

# Add the git remote (insert your token before the hostname)
git remote add xhost "https://${XHOST_TOKEN}@git.xhostd.com/<username>/my-site.git"

# Push
git push xhost master
```

App name rules (same as usernames, plus reserved prefix restrictions):
- Cannot be or start with: `git`, `api`, `www`, `admin`, `preview`, `staging`

## 5. Deploy and See It Live

After pushing, trigger the deploy:

```bash
SHA=$(git rev-parse HEAD)
curl -sf -X POST "${XHOST_API_URL}/apps/<app_id>/channels/<channel_id>/deploy" \
  -H "Authorization: Bearer $XHOST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sha":"'"$SHA"'"}'
```

Your site will be available at:

```
https://<app-name>-<username>.xhostd.com
```

For example, if your username is `alice` and your app is `my-site`, the URL is:
```
https://my-site-alice.xhostd.com
```

Check deployment status with:
```
/xhost:status
```

## 6. Preview Branches

To deploy a preview of a feature branch:

```
/xhost:preview
```

Or manually:

1. Create a wildcard channel (one-time setup per app):
   ```bash
   curl -sf -X POST "${XHOST_API_URL}/apps/<app_id>/channels" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"wildcard","git_ref_binding":"branch:*"}'
   ```

2. Push your feature branch and trigger a deploy:
   ```bash
   git push xhost my-feature
   SHA=$(git rev-parse HEAD)
   curl -sf -X POST "${XHOST_API_URL}/apps/<app_id>/channels/<channel_id>/deploy" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"sha":"'"$SHA"'"}'
   ```

3. The deploy fan-out creates a `preview-my-feature` child channel. After a few seconds, list channels to find the preview URL:
   ```bash
   curl -sf "${XHOST_API_URL}/apps/<app_id>/channels" \
     -H "Authorization: Bearer $XHOST_TOKEN"
   ```

Preview channels are cleaned up automatically when the branch is deleted from git.

## 7. Ongoing Deploys

To redeploy, push your changes and then trigger the deploy:

```bash
# Make changes, commit, and push
git add .
git commit -m "update homepage"
git push xhost master
```

Then trigger the deploy via the API, or use the shorthand:
```
/xhost:deploy
```

This stages any uncommitted changes, commits them, pushes, and triggers the deploy.

## Troubleshooting

**"Token not set" or "XHOST_TOKEN not set"**
Run `export XHOST_TOKEN=xh_...` with your token. For persistence, add it to your shell profile.

**"XHOST_API_URL not set"**
Run `export XHOST_API_URL=https://api.xhostd.com` (or your instance URL).

**"invite is invalid or already used"**
The invite code was already consumed or is incorrect. Ask for a new one.

**"username is already taken"**
Choose a different username. Usernames are globally unique.

**"app name is already taken"**
You already have an app with that name. Choose a different name or delete the existing app.

**"could not provision git backend"**
A server-side issue with the git backend. Try again or contact the operator.

**"xhost remote not found"**
The current git repo does not have an `xhost` remote. Run `/xhost:init` to set one up.

**Push rejected or authentication failure**
Make sure the git remote URL includes your token: `https://xh_...@git.xhostd.com/...`. You can update it with:
```bash
git remote set-url xhost "https://${XHOST_TOKEN}@git.xhostd.com/<username>/<app>.git"
```

**Site not updating after push**
The deploy pipeline runs asynchronously. Wait a few seconds and check status with `/xhost:status`. If the channel status shows `provisioning`, the deploy is still in progress. If it shows `running`, the deploy is complete.

**"cannot delete the prod channel"**
The production channel is permanent and cannot be deleted. Delete the entire app instead if needed.
