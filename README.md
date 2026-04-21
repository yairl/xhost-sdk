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

## Usage

Just use `/xhost` — it handles everything:

```
"deploy my website"          → signs up, creates app, pushes, deploys
"check my app status"        → shows apps, channels, URLs, deploy state
"create a preview for this branch" → pushes branch, creates preview URL
```

Or invoke it explicitly:

```
/xhost
```

The single `/xhost` skill handles account setup, app creation, deploys, previews, and status checks. Claude figures out what you need from context.

## What xhost supports

- **Static sites** — HTML/CSS/JS served by nginx
- **Node.js apps** — Express, Next.js, Fastify, Vite (provide `install.sh` + `launch.sh`)
- **Python apps** — FastAPI, Flask, Django (provide `install.sh` + `launch.sh`)
- **Any language** — if the runtime is in the base image, just write your scripts

## How it works

1. You push code to xhost's git server
2. You trigger a deploy (explicitly, via `/xhost` or the API)
3. xhost runs your `install.sh` (install deps) then `launch.sh` (start app on `$PORT`)
4. Your app is live over HTTPS with a wildcard cert

## Requirements

- A valid xhost token (`XHOST_TOKEN` env var)
- Git installed locally

## License

MIT
