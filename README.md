<div align="center">
  <a href="https://monetizationos.com">
  <img alt="MonetizationOS logo" src="https://app.monetizationos.com/static/monetizationos-logo.png" height="48">
  </a>
  <h1>MonetizationOS WordPress VIP Edge Worker Proxy</h1>
</div>

[MonetizationOS](https://monetizationos.com) powers monetization for human and bot users alike. Use this [WordPress VIP edge worker](https://github.com/Automattic/vip-edge-workers-sdk) to proxy your website and integrate MonetizationOS Surfaces, enabling seamless monetization experiences for sites served with static HTML.

This worker includes handling for both HTTP response modification and CSS-targeted Components for content modifications including: removal/truncation, displaying offerings, and custom messaging.

VIP edge workers are AssemblyScript compiled to WebAssembly, so this worker can't use [`@monetizationos/proxy`](https://www.npmjs.com/package/@monetizationos/proxy). It is a hand port of that pipeline, tracking 1.8.0, and behaves like our [Cloudflare proxy worker](https://github.com/MonetizationOS/cloudflare-proxy-worker). The intentional differences are listed by ID in [`DIVERGENCES.md`](DIVERGENCES.md). The SDK ships no host, so this repository also carries the local one the worker is tested on, in [`devkit/`](devkit).

Not affiliated with Automattic.

Read more about using MonetizationOS at [docs.monetizationos.com](https://docs.monetizationos.com).

## Getting Started

You need:

- A WordPress VIP environment that supports [Edge Workers](https://docs.wpvip.com/infrastructure/edge-servers/edge-workers/), and an authenticated [VIP-CLI](https://docs.wpvip.com/vip-cli/installing-vip-cli/).
- Node.js 22+.
- A MonetizationOS secret (`sk_`) key, and whole-body mode switched on for your organization. Without it, MonetizationOS answers 403 and pages are served unmodified.

Then build the worker:

```sh
npm install
npm run build
```

Deploy `build/mos-proxy/release.wasm` as your site's edge worker, with the [variables](#required-variables) below in its environment. See [Deployment](#deployment).

## Required Variables

The worker reads these with `getenv`. If any is missing, it passes every request through untouched:

- `MOS_SECRET_KEY`: Your MonetizationOS secret key. [Get your secret key](https://docs.monetizationos.com/docs/guides/environments/managing-environments#api-keys). It must be a secret `sk_` key, since only a secret key can send the page. A publishable key gets header, status and cookie decisions only (D9).
- `MOS_HOST`: MonetizationOS API host, e.g. `https://api.monetizationos.com`.
- `MOS_SURFACE_SLUG`: The slug for the MonetizationOS surface you want to target.

## Variables with Defaults

- `MOS_JWT_COOKIE`: Cookie name for authenticated user JWT sessions. Defaults to `mos_user_jwt`.
- `MOS_ANON_COOKIE`: Cookie name for anonymous sessions. Defaults to `mos_anon_id`.
- `MOS_ANON_FALLBACK`: Set to `false` to stop minting an anonymous id when the reader has none. Defaults to `true`.
- `MOS_ENDPOINTS_PREFIX`: Path prefix for proxied custom endpoints. Defaults to `/mos-endpoints/`.
- `MOS_ENVIRONMENT`: The environment path segment for custom endpoints. Derived from the key by default.

## Optional: paths that skip surface decisions

`MOS_IGNORE_PATTERNS` is a space-separated list of regular expressions, and `MOS_IGNORE_PREFIXES` a comma-separated list of path prefixes. Matching pathnames are served as cached, with no MonetizationOS call. A malformed pattern aborts page requests, since AssemblyScript can't catch it (D6).

WordPress paths that are never reader pages are always skipped: `/wp-admin/`, `/wp-login.php`, `/wp-json/`, `/xmlrpc.php`, `/wp-cron.php`, `/wp-includes/`, `/wp-content/`, and feeds.

## Optional: cookies forwarded to surface decisions

`MOS_SURFACE_DECISIONS_COOKIES` is a space-separated list of regular expressions. Cookie names matching any pattern are forwarded to the MonetizationOS surface-decisions API as `http.cookies`. Matching cookies are read from the incoming request `Cookie` header and from the origin response `Set-Cookie` headers; when the same name appears in both, the origin value is used. When unset or when no cookies match, `http.cookies` is omitted from the surface-decisions payload.

```sh
MOS_SURFACE_DECISIONS_COOKIES="^__session$ ^theme$ ^mos_"
```

Each pattern is a regex tested against the cookie **name**. Plain names like `^__session$` match exactly; prefixes like `^mos_` match any cookie whose name starts with `mos_`.

## Optional: authenticated routes

`MOS_AUTHENTICATED_ROUTES` is a comma-separated list of `METHOD /match>/apiPath` routes, POST, PUT or PATCH only. The worker merges the client's JSON body with the reader's identity and calls the API with `Authorization: PublicBearer <key>`. `POST /mos-api/offer-redemptions`, `/mos-api/access-checks`, `/mos-api/surface-decisions` and `/mos-api/counter-updates` are always added after the list.

## Optional: gradual rollout

`MOS_DECISIONS_PERCENT` is the share of readers, from 0 to 100, whose pages call MonetizationOS. The rest get the page as cached. Defaults to `100`. Each reader keeps their bucket through a random token in a `mos_bucket` cookie. Readers without cookies, such as most crawlers, are bucketed per request.

## Optional: resource fields

`buildResource` in [`assembly/resource.ts`](assembly/resource.ts) adds or overrides fields on the resource sent to MonetizationOS, over the default `{ id: path }`. Edit it for your site. Values are strings only.

## How it works

The worker runs in `onClientResponse`, after VIP's cache, and sends MonetizationOS the page itself:

```
onClientRequest   1a  /mos-endpoints/*      -> fetch API, respondWith  (short-circuit)
                  1b  authenticated routes  -> merge body + identity, respondWith
                  path not an asset or ignored -> accept-encoding: identity
                        |
              (cache, or origin on a miss: the page is stored as the origin sent it)
                        |
onClientResponse  text/html, not a redirect, path not ignored, uncompressed?
                                       no  -> served as is
                  POST /api/v1/surface-decisions  (application/vnd.mos.decision-stream.v1)
                       JSON line (http.proxyOrigin = { status }), \n, the page's raw bytes
                  answer: the decision's JSON line, then \n and the transformed page if one follows
                  origin status + headers
                    + anon cookie + surfaceBehavior.http ops
                    + surfaceBehavior.http.body if present (a workflow's own body),
                      else the page after the JSON line, else the page as served
                  cache-control: private, no-store
```

- **Caching.** VIP caches the origin's page and each reader's copy is transformed on the way out, so the origin renders a page once per cache lifetime. Every page view still makes one MonetizationOS call carrying the page.
- **Transforms.** MonetizationOS fills `resource.meta` from the page and applies `componentBehaviors` with the same transforms as the Cloudflare proxy, `replaceRange` included.
- **Identity.** A JWT cookie wins, then an existing anonymous id, else a new one is minted. Cookies the origin sets in its response win over the request's. The anonymous cookie is set when MonetizationOS minted the id, or for a JWT fallback MonetizationOS reports unauthenticated when the reader has no anonymous cookie.
- **Custom endpoints.** `<prefix>/rest?q` is proxied to the API with the client's method, headers and body, plus `x-mos-key: PublicBearer <key>` and `x-mos-endpoint-request-context`.
- **Version.** The `x-mos-proxy-version` header the worker sends is the core version it tracks.

### Failure handling

As with every MonetizationOS proxy, a failed decision serves the full page:

- **MonetizationOS fails:** the page as served, unmodified.
- **Not `text/html`, a redirect, compressed, or an ignored path:** as served, no MonetizationOS call.
- **Unconfigured:** pass-through.
- **A transform fails mid-page:** once the JSON line is out, MonetizationOS can't change the status, so the response body errors. The host fails the fetch and serves the page as it was.

## Commands

Every suite runs against the compiled binary rather than a re-implementation.

- `npm run build`: Build the worker to `build/mos-proxy/release.wasm`.
- `npm run web`: Build and serve the browser playground at http://localhost:8787.
- `npm test`: Unit-test the worker's logic over an explicit UTF-8 ABI.
- `npm run test:e2e`: Run the full pipeline against a fake API and a fake origin.
- `npm run test:web`: Run the bundled browser worker under a simulated Web Worker.
- `npm run test:wire`: Check the devkit's codec against the SDK's own, in both directions.
- `npm run test:hello`: Run the minimal example end to end.
- `npm run test:all`: Run all of the above.
- `npm run test:conformance`: Print the devkit's answers to the host probes, to diff against VIP.
- `npm run test:artifacts`: Build every release worker and report its size and startup memory.
- `npm run test:live`: Drive the playground in headless Chrome against a stand-in org. Set `CHROME_PATH` if Chrome isn't found.

`npm run artifact:report` inspects existing release artifacts without rebuilding them. Set `MAX_WASM_BYTES` and `MAX_WASM_MEMORY_BYTES` when running `npm run test:artifacts` to turn the report into a failing gate. The memory figures are the module's startup linear memory, not its peak.

## Local Development

1. Install dependencies:
    ```sh
    npm install
    ```
2. Start the playground:
    ```sh
    npm run web
    ```

To run against a real org, copy [`.env.local.example`](.env.local.example) to `.env.local`. See [`playground/`](playground). What the real VIP host is known to do, and the probe worker that checks it, are in [`tools/conformance/`](tools/conformance).

## Deployment

Build the worker:

```sh
npm run build
```

Then publish `build/mos-proxy/release.wasm` with VIP-CLI, following WordPress VIP's [Edge Workers documentation](https://docs.wpvip.com/infrastructure/edge-servers/edge-workers/). Test on a non-production environment first.

## Repository layout

```
assembly/           the worker (AssemblyScript)
DIVERGENCES.md      where it differs from @monetizationos/proxy, by ID
test/               unit, e2e, browser-bundle and live suites; wire/ pins the devkit codec to the SDK's
devkit/             the host half of the ABI in JS: msgpack, host functions, real lol-html
playground/         browser playground and local server, with an optional live pass-through
tools/conformance/  probe worker to deploy to VIP, and the devkit's answers to compare
examples/hello/     the smallest worker that exercises both phases
scripts/            asc build, playground bundle, artifact report, a worker as a local edge
```

[`AGENTS.md`](AGENTS.md) says where to look when a question is about the SDK or the real host.

## License

[MIT](LICENSE)
