# cf-deploy-redirect

Configure Cloudflare zones to redirect to a destination URL — correctly, idempotently, and with verification.

This is the tool for redirect-only domains. For everything else on Cloudflare, use Cloudflare's `cf` CLI (the replacement for wrangler). This script deliberately uses neither: it calls the API directly with `curl` and `jq` and a narrowly scoped token file, so it has no CLI to keep in step with and nothing in it needs broad account access.

For redirect-only domains: brand variants, typo-catchers, acquired names, retired products, anything you own defensively and want pointed at the real site. One command configures any number of zones and then proves the result over the wire.

```
$ cf-deploy-redirect --target https://example.com/ --apply

cf-deploy-redirect  →  https://example.com/

  oldbrand.com
    zone .................. found (abc12345…)
    A @ ................... already correct
    CNAME www ............. already correct
    redirect rule ......... set (301 → https://example.com/)
    always_use_https ...... set off
    hsts .................. off

waiting 20s for rule propagation…

Verification

  oldbrand.com
    http://@ .............. 301 → https://example.com/  [1 hop, cache-control: max-age=3600]
    https://@ ............. 301 → https://example.com/  [1 hop, cache-control: max-age=3600]
    http://www ............ 301 → https://example.com/  [1 hop, cache-control: max-age=3600]
    https://www ........... 301 → https://example.com/  [1 hop, cache-control: max-age=3600]

All good.
```

## Requirements

`bash` 3.2+, `curl`, `jq`. Nothing else — no Terraform, no Node, no Python. Works with the bash that ships on macOS.

Your zones must already exist in Cloudflare with nameservers delegated. This tool configures zones; it doesn't create them.

## Quick start

```bash
cp domains.txt.example domains.txt              # add your domains
cp cf-deploy-redirect.conf.example cf-deploy-redirect.conf   # set TARGET_URL
./cf-deploy-redirect                            # dry run — reads only
./cf-deploy-redirect --apply                    # write it
```

Dry run is the default. Nothing is ever written without `--apply`.

## What it configures, and why

Per zone:

| Setting | Value | Why |
|---|---|---|
| Apex `A` | `192.0.2.1`, proxied | RFC 5737 documentation address — guaranteed unroutable. The edge intercepts before any origin fetch, so nothing is ever served from it. |
| `www` `CNAME` | the apex, proxied | One place to change the target. Publicly identical to an A record, since proxying means Cloudflare answers with its own IPs either way. |
| Redirect rule | `301` to your target | Host-scoped to exactly the apex and `www` — see below. Every path goes to the target as-is, unless `--preserve-path`. |
| Always Use HTTPS | **off** | It fires *before* redirect rules, so leaving it on makes `http://` take two hops. Off, the redirect rule catches plain HTTP directly. Nothing is served from these hostnames, so there's no content to protect by forcing HTTPS first. |
| `Cache-Control` | optional | Bounds how long browsers pin the redirect. See below. |
| HSTS | **reported, never changed** | Disabling it doesn't un-pin browsers that already cached the policy. That's a human decision. |

### Host-scoped rules, not match-all

The redirect rule's expression names the two hostnames explicitly:

```
http.host in {"oldbrand.com" "www.oldbrand.com"}
```

rather than matching all requests. This matters more than it looks. A match-all rule redirects *any* subdomain you later add to that zone — and because browsers cache 301s indefinitely, it seeds permanently-cached redirects for the exact hostname you're trying to launch. Scoping the rule means `app.oldbrand.com` just works when you create it.

For the same reason the tool creates only apex and `www` records, never a wildcard. Every other subdomain stays clean.

### Preserving paths

By default every request lands on the target URL exactly — `oldbrand.com/pricing` goes to `https://example.com/`. That's right for brand variants and typo domains, which never had pages of their own.

For a **retired or renamed site** whose pages exist at the new address, use `--preserve-path` (or `PRESERVE_PATH=true`):

```
oldsite.com/pricing?ref=x  →  https://newsite.com/pricing?ref=x
```

The rule's target becomes an expression, `concat("https://newsite.com", http.request.uri.path)`, and the query string is kept as always. The target must be a bare origin — `https://newsite.com/`, no path or query — and the tool refuses anything else rather than build a URL with a doubled path. Verification adds a `path+query` check that requests a deep URL and confirms it arrives intact.

The flag applies to every domain in the run. To mix modes, run the tool twice with different domain lists.

### Bounding the cache

A `301` with no `Cache-Control` is heuristically cacheable **indefinitely** — the browser stops asking and rewrites the URL locally, forever. There is no server-side purge for that; Cloudflare's "Purge Everything" doesn't touch it.

Setting `CACHE_CONTROL=max-age=3600` (or `--cache-control`) adds a response-header rule that bounds it to an hour. This costs nothing in SEO — search engines key off the status code, not the freshness header — and it keeps the hostname reusable. Worth doing before a domain sees any traffic; afterwards it only helps future visitors.

If you might genuinely want a hostname back later, also consider `REDIRECT_STATUS=302`.

## Safety model

- **Dry run by default.** `--apply` is the only thing that writes.
- **Refuses zones it doesn't own.** `PUT` to a Cloudflare phase entrypoint replaces *every* rule in that phase. The tool reads first and aborts any zone holding rules it didn't create, identified by a `ref` marker. Inspect them with `--dump-rules`; authorize the overwrite with `--replace-rules`.
- **Detects DNS conflicts.** A `CNAME` can't coexist with an `A` at the same name. The tool reports the conflict instead of failing on an opaque API error.
- **Leaves other record types alone.** Only `A`/`AAAA`/`CNAME` are inspected. `MX`, `TXT`, SPF, DKIM, and DMARC are untouched, so a redirect domain that also receives email keeps working.
- **Idempotent.** Re-running reports `already correct` and writes nothing.
- **Verifies over the wire.** Every hostname is checked on both schemes after the fact — status, hop count, final URL, and `Cache-Control`.

## The token

Never pass a token on the command line. Resolution order:

1. `$CLOUDFLARE_API_TOKEN`
2. `--token-file PATH`, or `~/.config/cloudflare/cf-deploy-redirect-token` if it exists — **must be mode 600**, the tool refuses anything looser
3. Interactive prompt, hidden input

```bash
mkdir -p ~/.config/cloudflare && (umask 077; cat > ~/.config/cloudflare/cf-deploy-redirect-token)
```

`umask 077` creates it `600` from the start rather than briefly world-readable.

The token is kept out of `ps` output (fed to curl via a config file on stdin, not `-H`) and out of `bash -x` traces (xtrace is suspended around each call).

**Scopes** — all zone-level:

| Permission | Level |
|---|---|
| Zone → Zone | Read |
| Zone → DNS | Edit |
| Zone → Single Redirect | Edit |
| Zone → Zone Settings | Edit |
| Zone → Transform Rules | Edit *(only with `--cache-control`)* |

Scope the token to **specific zones**, not "All zones from an account". That structurally prevents it touching anything else, which matters more than it sounds when the tool's job is bulk configuration.

In Cloudflare's account-owned token UI the permission groups are named `<Thing> Read` / `<Thing> Write` rather than the `Zone → X → Edit` phrasing above — search `DNS`, `Redirect`, `Settings`. Account-owned tokens also require Super Administrator on the account.

## Options

```
--target URL          Default destination
--domains FILE        Domain list (default: ./domains.txt)
--config FILE         Config file (default: ./cf-deploy-redirect.conf)
--token-file PATH     Token file, mode 600
--status N            301, 302, 307 or 308 (default: 301)
--preserve-path       Keep the request path; target must be a bare origin
--rule-ref REF        Ownership marker (default: cf_deploy_redirect)
--cache-control VAL   e.g. max-age=3600
--placeholder-ip IP   Origin placeholder (default: 192.0.2.1)
--propagation-wait N  Seconds before verifying after --apply (default: 20)
--apply               Write changes
--replace-rules       Permit overwriting foreign rules
--dump-rules          Print existing rules, read-only
--verify-only         No API calls; just check live behavior
--no-verify           Skip verification
-h, --help
```

Precedence: flags → config file → built-in defaults.

Domains can also be passed as arguments, which overrides the domain file entirely:

```bash
./cf-deploy-redirect --target https://example.com/ oldbrand.com oldbrand.net
```

## Gotchas

Things that cost real debugging time.

**An HTTPS-upgrade rule alone is a dead end.** Cloudflare's stock "Redirect from HTTP to HTTPS" template rewrites the scheme and nothing else. On a zone whose origin is an unroutable placeholder, that means `http://` upgrades correctly and then `https://` falls through to origin and returns **522** after ~20 seconds. A redirect-only zone needs a rule that actually names a destination. If you're adopting zones someone else set up, check for this — it fails silently and looks configured.

**The phase entrypoint endpoint rejects `kind` and `phase`.** `PUT /zones/{id}/rulesets/phases/{phase}/entrypoint` accepts only `name`, `description`, and `rules`; the rest is implied by the URL path. Sending them gives `400 invalid JSON: unknown field "kind"`. Cloudflare's documented example is for `POST /zones/{id}/rulesets`, which is a different endpoint and does require them.

**Don't verify immediately after writing.** Different PoPs answer from different rule versions for a few seconds, so a zone checked right after its own `PUT` reports false failures. This tool verifies in a separate pass after `--propagation-wait`.

**On a brand-new zone, HTTPS may fail while HTTP succeeds.** That's Universal SSL still being issued, not a rule problem. Check the certificate's `notBefore` before debugging anything:

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -dates
```

Minutes old means wait and re-verify.

**Response-header transforms do reach redirect responses.** This was not obvious — a redirect short-circuits before origin, so it was unclear whether the response phase ran at all. It does, on both HTTP and HTTPS, which is why bounding the cache needs no Worker.
