# Integration of SEO4Ajax in Salesforce Storefront Next

[SEO4Ajax](https://www.seo4ajax.com) is a service that allows AJAX websites
(e.g. based on React, Vue.js, Svelte, Angular, Backbone, Ember, jQuery etc.) to
be indexable by search engines, social and ad networks.

This file describes how to integrate SEO4Ajax in a
[Storefront Next](https://developer.salesforce.com/docs/commerce/sfnext/overview)
storefront, the React storefront framework for Salesforce B2C Commerce, through
a middleware registered on the root route.

## How it works

Storefront Next runs on React Router 7, and a middleware registered in the root
route's `middleware` array runs on every request the storefront serves. A
middleware that returns a `Response` without calling `next()` replaces the
storefront's own response, which is exactly what dynamic rendering needs.

```
                            ┌── crawler ──────► api.seo4ajax.com ──► prerendered HTML
GET /any/path ──► root         middleware
                  array     └── everyone else ─► the storefront renders as usual
```

The whole site is covered, not a list of routes: product, category, search,
content and custom pages all pass through the same middleware. There is no
per-page configuration, and adding a route later needs no change here.

Three kinds of visitor matter, and they are three rather than two:

| Who | Receives | Why |
|---|---|---|
| Search engine crawlers, social and ad networks | the prerendered snapshot | they will not wait for client-side rendering |
| Real visitors | the storefront's own response | interactive, always current |
| The SEO4Ajax scraper (`s4a/1.0`) | the storefront's own response | it is the thing producing the snapshots; serving it one would recurse |

## Requirements

- A Storefront Next project. The code below was written against Storefront Next
  1.3.1 (React Router 7.18, React 19.2).
- The site token from the site settings page in the console of SEO4Ajax
  (e.g. `12918fa3c2e945aaf4625a81c93a3181`).

## Instructions

1. Add the site token to your environment, as `SEO4AJAX_SITE_TOKEN`:

   ```sh
   # .env — no PUBLIC__ prefix, so it stays server-side and never reaches the browser
   SEO4AJAX_SITE_TOKEN=12918fa3c2e945aaf4625a81c93a3181
   ```

   On Managed Runtime, set it as an environment variable on the environment.
   Note that MRT environment variable values cannot contain a slash.

2. Create `src/middlewares/seo4ajax.server.ts` and paste the code below.

3. Register it in the root route's middleware array, in `src/root.tsx`:

   ```ts
   import { seo4ajaxMiddleware } from '@/middlewares/seo4ajax.server';

   export const middleware: MiddlewareFunction<Response>[] = [
       correlationMiddleware,
       errorCacheControlMiddleware,
       requestOriginMiddleware,
       loggingMiddleware,
       seo4ajaxMiddleware,
       // … the rest of the storefront's middleware
   ];
   ```

   Position it **before** the authentication, basket and commerce-API
   middleware, as above. Everything registered after it is skipped whenever a
   snapshot is returned, so a crawler request costs one call to SEO4Ajax instead
   of a guest login and a series of SCAPI queries.

4. Check the result, with and without a crawler user agent:

   ```sh
   curl -A "Googlebot" https://<host>/<path>   # the prerendered snapshot
   curl https://<host>/<path>                  # the storefront
   ```

   The two responses should differ in size, and the crawler response should
   carry `content-type: text/html; charset=utf-8` and `cache-control: no-store`.

## Code

```ts
import type { MiddlewareFunction } from 'react-router';

const API_URL = 'https://api.seo4ajax.com/';

const USER_AGENT_TEST =
    /google|bot|-user|meta-external|lighthouse|spider|pinterest|crawler|archiver|flipboardproxy|mediapartners|facebookexternalhit|insights|quora|whatsapp|slurp/i;
const FILENAME_EXTENSION_TEST = /\.(?!htm)[^.]+$/;

// SEO4Ajax's own scraper, which renders the page and must therefore always
// receive the storefront's response, or prerendering would recurse into itself.
const SCRAPER_TEST = /\bs4a\//i;

// A cold render must not blank the page for a real crawler: give up and let the
// storefront answer instead.
const API_TIMEOUT_MS = 5000;

function isCrawler(userAgent: string | null): boolean {
    return !!userAgent && !SCRAPER_TEST.test(userAgent) && USER_AGENT_TEST.test(userAgent);
}

// Let SEO4Ajax see which crawler this is and where it came from.
function forwardedHeaders(request: Request): Headers {
    const headers = new Headers();
    const userAgent = request.headers.get('user-agent');
    const forwardedFor = request.headers.get('x-forwarded-for');

    if (userAgent) headers.set('user-agent', userAgent);
    if (forwardedFor) headers.set('x-forwarded-for', forwardedFor);

    return headers;
}

// Returns the snapshot, or null whenever the storefront should answer instead.
async function fetchSnapshot(request: Request): Promise<Response | null> {
    const siteToken = process.env.SEO4AJAX_SITE_TOKEN;

    if (!siteToken) {
        console.error('[seo4ajax] SEO4AJAX_SITE_TOKEN is not set');
        return null;
    }

    const url = new URL(request.url);
    const target = API_URL + siteToken + url.pathname + url.search;

    try {
        const response = await fetch(target, {
            headers: forwardedHeaders(request),
            // Hand a prerendered redirect to the crawler rather than following it here.
            redirect: 'manual',
            signal: AbortSignal.timeout(API_TIMEOUT_MS),
        });

        if (response.status >= 400) {
            console.warn(`[seo4ajax] ${response.status} from ${target}`);
            return null;
        }

        const headers = new Headers({ 'content-type': 'text/html; charset=utf-8' });

        // The snapshot is crawler-specific and the CDN in front of Managed
        // Runtime keys on the URL alone, so it must never enter a shared cache.
        headers.set('cache-control', 'no-store');

        const location = response.headers.get('location');
        if (location) headers.set('location', location);

        return new Response(await response.text(), { status: response.status, headers });
    } catch (error) {
        console.warn(`[seo4ajax] ${(error as Error).message} while fetching ${target}`);
        return null;
    }
}

export const seo4ajaxMiddleware: MiddlewareFunction<Response> = async ({ request }, next) => {
    if (request.method !== 'GET') {
        return next();
    }

    const url = new URL(request.url);

    // Skip anything that is not a document: static assets, and React Router's
    // own `<route>.data` single-fetch URLs, which only a browser requests.
    if (FILENAME_EXTENSION_TEST.test(url.pathname)) {
        return next();
    }

    if (!isCrawler(request.headers.get('user-agent'))) {
        return next();
    }

    // Returning a Response without calling next() replaces the storefront
    // response outright, and skips every middleware registered after this one.
    return (await fetchSnapshot(request)) ?? next();
};
```

## Notes

**Middleware, not a loader.** A route loader cannot serve the snapshot, in
either of the two ways it might appear to. Returning a `Response` from a loader
does not short-circuit anything: the response is serialized into the RSC payload
and embedded in the rendered page, so the crawler receives the storefront shell
with the entire snapshot inlined inside `<script>` tags. Throwing a `Response`
is worse — a thrown non-redirect `Response` is an error in React Router, so it
lands in the error boundary. Only middleware replaces the response.

**Failing open.** Any timeout, network error or status of 400 or above returns
`null` and the storefront answers normally. A prerendering outage, or a path
that has not been configured in SEO4Ajax yet, must never blank the page for a
crawler.

**URL prefixes.** Storefront Next can serve locale- and site-prefixed URLs
(`app.url.prefix`, e.g. `/:siteId/:localeId`). The middleware sends
`url.pathname` verbatim, which is the URL the crawler actually requested, so the
snapshot key includes the prefix: `/global/en-GB/some-page`, not `/some-page`.
Configure the paths in SEO4Ajax accordingly, one set per locale you publish.

**Caching.** Managed Runtime sits behind a CDN that keys on the URL alone, so
there is no way to vary a cached document by user agent — hence
`cache-control: no-store` on the snapshot. The consequence is that a page
already in the CDN cache never reaches the middleware, because the origin is not
invoked on a cache hit. Pages you cache aggressively are therefore not covered
by dynamic rendering. If your catalogue is cached and its content is
client-rendered, run the
[SEO4Ajax Cloudflare Worker](https://github.com/seo4ajax/seo4ajax-cloudflare) on
a CDN in front of the storefront instead, where the decision happens before the
cache.

**Security headers.** Middleware registered after `securityHeadersMiddleware`
has that middleware's headers applied to the snapshot on the way out; registered
before it, the snapshot is returned as it came from SEO4Ajax. Placing it early,
as in step 3, is the cheaper option and leaves the prerendered document exactly
as SEO4Ajax produced it.

**Query parameters.** The middleware forwards `url.search` unchanged, so each
combination of parameters is a distinct URL that SEO4Ajax prerenders separately.
Two consequences are worth knowing. Storefront Next builds canonical URLs from
an allowlist of query parameters and strips everything else, so parameters that
change page content should be added to that allowlist
(`src/utils/canonical-url.ts`). And crawlers still need ordinary `<a href>`
links to discover parameterised URLs. Values kept in the fragment (`#…`) cannot
be prerendered, since a fragment never reaches the server.

**Discovery.** Storefront Next generates a sitemap from the commerce catalogue.
Pages whose content is produced by client-side code, and parameterised URLs, are
not in it; add them to a sitemap of your own if you want them crawled.

**Client-rendered widgets.** Two things reliably break third-party widgets on
this stack, both unrelated to SEO4Ajax but usually discovered at the same time.

The first is the Content-Security-Policy. Storefront Next ships a strict CSP
with no `'unsafe-inline'` in `script-src`, relying on a per-request nonce. A
nonce only applies to tags the browser parsed from the document, so anything a
vendor's bootstrap injects at runtime can only be admitted by origin. Add the
vendor's asset origin to `script-src` and `style-src`, and its API origin —
often a different host — to `connect-src`, by spreading `defaultCspDirectives`
in `config.server.ts`. Each directive you set replaces the default entirely, so
spread it or you will drop `'self'`.

The second is hydration. A `<script>` tag in a route's JSX executes while the
browser parses the document, which is before React hydrates; by then the widget
has filled its container, React finds child elements in a node it rendered
empty, and aborts hydration — and its recovery, re-rendering the tree, deletes
what the widget built. The symptom is that the widget appears for a moment and
then vanishes. `suppressHydrationWarning` does not fix it: that covers differing
attributes and text on one element, never extra child elements. Load third-party
widgets from a `useEffect` instead, so nothing touches the DOM until hydration
has finished.

**During development.** `pnpm dev` serves the storefront on `localhost`, which
SEO4Ajax cannot reach to take a snapshot of, and no SEO4Ajax site is configured
for a localhost path. Expose the dev server through a tunnel and register that
hostname as a site, remembering that Vite rejects requests whose `Host` it does
not recognise — add the tunnel domain to `server.allowedHosts` in
`vite.config.ts` or every request answers 403. Tunnels that add
`x-forwarded-proto: https` also change the origin the storefront computes for
itself, which can invalidate the SLAS redirect URI and break guest login; pin
the hostname with `EXTERNAL_DOMAIN_NAME`, or register the tunnel's callback URL
with your SLAS client.

## Other Salesforce storefronts

This file covers Storefront Next. The same result on other stacks:

- **SFRA** — a cartridge that intercepts a controller route with
  `server.prepend`. Note that B2C Commerce's page cache sits in front of the
  application server, so pages served from cache never reach the cartridge.
- **Any B2C Commerce storefront, site-wide** — run the
  [SEO4Ajax Cloudflare Worker](https://github.com/seo4ajax/seo4ajax-cloudflare)
  on your own CDN in front of the storefront. Salesforce supports a
  customer-managed CDN or reverse proxy in front of B2C Commerce; the embedded
  CDN does not expose customer Cloudflare Workers.
