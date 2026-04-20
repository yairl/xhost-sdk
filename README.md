# xhost SDK

Claude Code plugin for [xhost](https://xhostd.com) — deploy static sites and dynamic applications with a single token. Push code, get HTTPS URLs.

## Install

```
/plugin marketplace add yairl/xhost-sdk
/plugin install xhost@xhost-sdk
```

After installing, reload plugins in your current session:

```
/reload-plugins
```

## Skills

| Skill | Description |
|-------|-------------|
| `/xhost:init` | Create a new xhost app and connect the current project |
| `/xhost:deploy` | Push code and deploy to production |
| `/xhost:status` | Check app status, channels, and recent deploys |
| `/xhost:preview` | Create a preview channel for a branch |
| `/xhost:login` | Authenticate with an existing xhost token |
| `/xhost:signup` | Create a new xhost account |
| `/xhost:xhost` | General xhost knowledge and API reference |

## Quick Start

```
# In a project directory with your code:
/xhost:signup          # Create an account (first time only)
/xhost:init            # Create an app and connect this project
/xhost:deploy          # Push and deploy — get your HTTPS URL
```

That's it. Your site is live at `https://<app>-<user>.xhostd.com`.

## What xhost supports

- **Static sites** — HTML/CSS/JS served by nginx
- **Node.js apps** — Express, Next.js, Fastify, Vite (provide `install.sh` + `launch.sh`)
- **Python apps** — FastAPI, Flask, Django (provide `install.sh` + `launch.sh`)
- **Any language** — if the runtime is in the base image, just write your scripts

## How it works

1. You push code to xhost's git server
2. You trigger a deploy (explicitly, via `/xhost:deploy`)
3. xhost runs your `install.sh` (install deps) then `launch.sh` (start app on `$PORT`)
4. Your app is live over HTTPS with a wildcard cert

## Requirements

- A valid xhost token (`XHOST_TOKEN` env var)
- Git installed locally

## License

MIT
