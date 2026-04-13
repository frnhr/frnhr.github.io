# Obsidian Vault Editing from Claude Code Cloud

Pattern for editing an Obsidian vault from Claude Code on the Web — web UI or mobile — while the Mac is off, using Obsidian Sync as the single source of truth.

> Yes, that's two [em-dashes](https://www.youtube.com/shorts/XKsPaX2NVOs) in this intro! 

## The What?

Wanna be able to spin up a Claude Code remote session and have it access my Obsidian vault.

## Why?

Yes.

## What's the Problem?

Cloud sessions need a git repo to clone. The vault isn't in git, and shouldn't be. Because I don't want it in git. Because I'm using Obsidian Sync already. Because I don't want to mess around with troubleshooting the sync-via-git from mobile. Because :scream:

## Sooo...

So, we need to sync the vault on a remote environment.

## Doesn't Obsidian have a CLI?

Well, yes, and no. The CLI that comes with the Obisidian app needs the Obsidian app to be running. But there is also [obsidian-headless](https://github.com/obsidianmd/obsidian-headless) that does exactly what we need.

## So, it just works!

BUAAHAHAAAHA HAA HAA HAA :D :) :\| :'( :'O

## Ok can you cut the crap and get to it already?

Sure! Claude-generated content below.

Trigger warning, though: 100% unsupervised patching of Node internals to work with a HTTP proxy! I am looking away, for sure!

Anyways:

## The Crap, Cut

A minimal **bootstrap repo**, separate from the vault, containing only cloud-session wiring:

```
SessionStart → ob sync --path ./vault    (blocking pull)
Claude works on ./vault/
Stop         → ob sync --path ./vault    (blocking push)
```

[`obsidian-headless`](https://github.com/obsidianmd/obsidian-headless) (official npm package) does the sync — Node CLI, no desktop app, works on Linux. Credentials are injected via per-repo env vars in the cloud session config, which sync to mobile automatically.

## The Proxy Problem

Claude Code cloud sandboxes route all traffic through an HTTPS proxy (`HTTPS_PROXY` is set). Node 22's built-in `fetch` and `WebSocket` both use an internal bundled undici that **ignores `HTTPS_PROXY` entirely** and cannot be patched from user-land. The dispatcher lives at `node:internal/deps/undici/undici` — not the public `undici` package, a separate instance.

`ob login` fails at DNS resolution (`EAI_AGAIN`). `ob sync` connects but hangs indefinitely on the WebSocket.

## Fix

A preload script, loaded via `NODE_OPTIONS=--require`, replaces both globals before `ob` loads:

```js
// fetch — npm undici with ProxyAgent
const { ProxyAgent, setGlobalDispatcher, fetch: undiciFetch } = require(root + '/undici');
const dispatcher = new ProxyAgent(proxyUrl);
setGlobalDispatcher(dispatcher);
globalThis.fetch = (url, opts) => undiciFetch(url, { ...opts, dispatcher });

// WebSocket — ws@8 + https-proxy-agent@5 (v6+ is ESM-only)
const WS = require(root + '/ws');
const { HttpsProxyAgent } = require(root + '/https-proxy-agent');
globalThis.WebSocket = class ProxiedWebSocket extends EventTarget { ... }
```

`obsidian-headless` captures `var gr = WebSocket` at module load time, so replacing the global before `require('/path/to/ob')` is enough — no patching of the module itself.

The WebSocket shim translates between `ws@8`'s Node-style event emitter API and the browser WebSocket API that `obsidian-headless` expects: `isBinary` flag for text/binary frame distinction, `Object.assign(new Event('close'), { code, reason })` in lieu of `CloseEvent` (not available in Node), and direct `on*` property setters.

> / The Crap, Cut

## OMG How Can You Live With This?!

I COULDN'T! So I asked Claude the magic question: 

> What version of Node are we using? Is there a newer version that supports HTTP_PROXY?

## Aaand?

🦄🌈✨🦄✨🌈✨🦄🌈✨🦄🦄

v22.19.0+


## Other Surprises

`ob` CLI flags differ from what the README implied at time of writing:

| Wrong | Correct |
|---|---|
| `ob sync-setup --vault <local> --remote-vault <name>` | `ob sync-setup --path <local> --vault <name>` |
| `ob sync-setup --vault-password` | `ob sync-setup --password` |
| `ob sync --vault` | `ob sync --path` |

Auth token lives at `~/.config/obsidian-headless/auth_token`, not `~/.obsidian-headless/config.json`.

## What now?

Nothing, it works. Here, touch some grass 🌱🍀🌾🌾🪲🌱

Or: https://github.com/frnhr/vault-cloud-bootstrap

