# Wikimedia User-Agent policy

All requests to `*.wikipedia.org`, `*.wikidata.org`, `commons.wikimedia.org`, `*.wikivoyage.org`, and `upload.wikimedia.org` MUST send a descriptive User-Agent identifying the application and a contact address. Wikimedia's policy: <https://meta.wikimedia.org/wiki/User-Agent_policy>.

## Exact UA string

```
myfootmarks/0.0.1 (graner@gmail.com)
```

The version segment tracks `.claude-plugin/plugin.json`'s `version` field. When the plugin version changes, this UA string must change with it (the plugin's release-time sync script will eventually own this; until then, update by hand).

## Sending the UA

In Bash via curl:

```bash
curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" "$URL"
```

In WebFetch invocations, pass the UA as part of the prompt or as an explicit header when the tool supports it. When in doubt, prefer Bash + curl for Wikimedia endpoints — the UA is non-negotiable.

## Why a generic curl UA is not sufficient

Wikimedia rejects (HTTP 403) or rate-limits requests with blank, generic, or library-default UAs. The descriptive UA above is what unblocks 200 OK responses at the volumes a single trip's resolver run produces (~50 entities × ~3 sources average = ~150 fetches).

## Non-Wikimedia endpoints

- `api.openverse.org` — no UA requirement; sending the same UA is harmless.
- Venue homepages (for og:image) — send a browser-like UA (`Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.0 Safari/605.1.15` or similar) to avoid bot blocks. Add `Referer: https://www.google.com/` to mimic a search referral.
