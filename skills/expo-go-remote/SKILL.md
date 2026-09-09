---
name: expo-go-remote
description: Connect Expo Go on a remote device (outside the LAN) to a local Expo dev server using an ephemeral Cloudflare quick tunnel. Use when the user wants to open Expo Go from outside, needs a public URL for Metro, reports that Expo Go cannot connect or fails to fetch the bundle, or mentions exp://, trycloudflare, or tunneling Expo.
version: 1.0.0
license: MIT
---

# Expo Go remote access via Cloudflare quick tunnel

Expose a local Expo (Metro) dev server through an ephemeral Cloudflare quick
tunnel so Expo Go on a phone outside the LAN can load the app. Works for any
Expo project; no app changes required.

## Why ports must align (read this first)

Expo CLI builds the manifest's URLs as `<request-scheme>://<request-host>:<metro-port>`.
Behind a tunnel the request host is the tunnel domain, but Metro's listen port
is appended verbatim. Cloudflare's edge only serves ports **80** and **443**.

| Expo Go entry | How Expo Go fetches the manifest | Metro must listen on | Tunnel target |
| --- | --- | --- | --- |
| `exp://<domain>` (recommended) | plain HTTP, port 80 | **80** | `http://localhost:80` |
| `https://<domain>` | HTTPS, port 443 | **443** | `http://localhost:443` |

Metro on its default port 8081 behind a tunnel **always fails**: the manifest
advertises `<domain>:8081`, which Cloudflare refuses. If the dev server is
already running on 8081, restart it on the aligned port (tell the user you are
restarting it).

## Workflow

1. **Find the dev server and its port.** Default Metro port is 8081. If it is
   running on the wrong port, stop it and start it aligned:
   - `npx expo start --port 80` (for the `exp://` flow), or
   - `npx expo start --port 443` (for the `https://` flow).
2. **Check cloudflared** (`cloudflared --version`). If missing, install it
   (e.g. `winget install Cloudflare.cloudflared`, `brew install cloudflared`,
   or download the binary from Cloudflare).
3. **Start a quick tunnel, detached**, logging to files so the URL can be
   parsed. Plain HTTP target that matches the chosen flow:
   `cloudflared tunnel --url http://localhost:80 --no-autoupdate`
   - Run it in the background and redirect stdout/stderr to temp files; the
     `https://<words>.trycloudflare.com` URL appears in the log within ~10s.
   - On Windows/PowerShell: `Start-Process cloudflared -ArgumentList ...`
     with `-RedirectStandardOutput`/`-RedirectStandardError`, then poll the
     log with `Select-String 'trycloudflare'`.
4. **Verify through the tunnel before telling the user it works:**
   - `curl http://<domain>/status` must return `packager-status:running`.
   - Manifest check (optional but catches the port bug): request the manifest
     the way Expo Go does and confirm the advertised URLs use `:80` (or
     `:443`), never `:8081`:
     ```bash
     curl -s http://<domain>/ \
       -H 'expo-platform: android' \
       -H 'expo-runtime-version: exposdk:<SDK>.0.0' \
       -H 'accept: multipart/mixed' | grep -o '"url":"[^"]\{0,80\}'
     ```
     `<SDK>` is the project's Expo SDK major version (see `app.json`). The
     response is `multipart/mixed`; the `launchAsset.url`, `hostUri`, and
     `debuggerHost` fields show what Expo Go will dial.
   - Bundle check: `curl -sI` the advertised `entry.bundle?...` URL and
     expect HTTP 200 (first compile may take a while; 10-20 MB is normal).
5. **Give the user the URL** for Expo Go → *Enter URL manually*:
   - `exp://<domain>` for the port-80 flow, or
   - `https://<domain>` for the port-443 flow.
   Tell them to use the exact scheme: `exp://` and `https://` are not
   interchangeable for an externally served domain.

## App backend env (optional, only if the app calls its own backend)

`EXPO_PUBLIC_*` variables are **baked into the bundle when Metro serves it**,
so changing `.env` requires an Expo restart to take effect on the device.

To make the backend reachable from outside the LAN, run a **second** quick
tunnel for the backend port and point the client env at its domain (each quick
tunnel process gets its own random domain):

```
EXPO_PUBLIC_API_SCHEME=https
EXPO_PUBLIC_API_HOST=<backend-tunnel-domain>
EXPO_PUBLIC_API_PORT=
EXPO_PUBLIC_WS_SCHEME=wss
EXPO_PUBLIC_WS_PORT=
```

Empty ports resolve to 443 with `https`/`wss`. The backend must serve both
HTTP API and WebSocket through that tunnel (quick tunnels support WebSocket).

## Gotchas

- Quick tunnels are **ephemeral**: a new random domain on every run, and the
  URL dies with the cloudflared process (machine sleep/reboot too). Treat
  every re-setup as producing a fresh URL.
- WebSocket (Metro HMR, logs, debugger) works through quick tunnels.
- If the phone is on the same LAN, plain `exp://<lan-ip>:8081` needs no tunnel
  and is faster; use the tunnel only for true remote access.
- Do not kill or reconfigure anything else (Docker, sandboxes, other tunnels)
  unless the user asks; this skill is only about exposing the dev server.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Manifest loads but bundle fetch times out | Manifest advertises `:8081` (Metro not aligned to 80/443) | Restart Metro on the aligned port; re-verify manifest URLs |
| `exp://` does nothing in Expo Go | Metro is on 443 but user entered `exp://` (or vice versa) | Match scheme to flow: `exp://` + port 80, `https://` + port 443 |
| Tunnel domain returns 530/1033 or hangs | cloudflared process died | Restart the tunnel; hand the user the new domain |
| Env changes have no effect in the app | `EXPO_PUBLIC_*` baked at serve time | Restart Expo, reload in Expo Go |
| Manifest check returns 400 | Wrong `expo-runtime-version` header | Use `exposdk:<SDK major>.0.0` from the project's `app.json` |
