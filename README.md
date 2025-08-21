# Tasking Manager Proxy (Cloudflare Worker)

Purpose: Call https://tasking-manager-production-api.hotosm.org from browser without CORS issues.

Note: In the default wildcard form anyone on the internet can use your proxy. Consider restricting it (whitelisting your site origins) to prevent abuse.

## Steps

1. Cloudflare Dashboard > Workers & Pages > Create > Worker > Hello World
2. Replace code with:

```javascript
const UPSTREAM = 'https://tasking-manager-production-api.hotosm.org';

export default {
  async fetch(request) {
    const u = new URL(request.url);
    const upstreamUrl = new URL(u.pathname + u.search, UPSTREAM);

    if (request.method === 'OPTIONS') {
      return new Response(null, {
        status: 204,
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET,HEAD,POST,PUT,PATCH,DELETE,OPTIONS',
          'Access-Control-Allow-Headers': request.headers.get('Access-Control-Request-Headers') || 'Content-Type,Authorization,Accept',
          'Access-Control-Max-Age': '86400'
        }
      });
    }

    const r = await fetch(upstreamUrl.toString(), {
      method: request.method,
      headers: request.headers,
      body: ['GET','HEAD'].includes(request.method) ? undefined : request.body,
      redirect: 'manual'
    });

    const h = new Headers(r.headers);
    h.set('Access-Control-Allow-Origin', '*');
    return new Response(r.body, { status: r.status, statusText: r.statusText, headers: h });
  }
};
```

3. Save and deploy
4. Copy the workers.dev URL
5. In frontend set:
```javascript
const TASKING_MANAGER_API_URL = 'https://YOUR_WORKER_SUBDOMAIN.workers.dev';
```
6. Test:
```javascript
fetch(TASKING_MANAGER_API_URL + '/api/v2/projects/').then(r=>r.json()).then(console.log);
```

## Restrict usage (recommended)

Replace wildcard with allow list:
```javascript
const origin = request.headers.get('Origin');
const allow = ['https://yourapp.com','http://localhost:5173'];
if (origin && allow.includes(origin)) {
  h.set('Access-Control-Allow-Origin', origin);
} else {
  h.set('Access-Control-Allow-Origin', 'https://yourapp.com');
}
```

Optional shared secret:
Frontend add header:
```javascript
{x-proxy-key: 'SECRET123'}
```
Worker before fetch:
```javascript
if (request.headers.get('x-proxy-key') !== 'SECRET123') return new Response('Forbidden', { status: 403 });
```

## CLI (optional)

```bash
npm install -g wrangler
mkdir tm-proxy && cd tm-proxy
wrangler init --yes
```

Put file src/worker.js with same code.

wrangler.toml:
```toml
name = "tm-wildcard-proxy"
main = "src/worker.js"
compatibility_date = "2024-08-21"
```

Deploy:
```bash
wrangler login
wrangler deploy
```

## Optional GET cache

```javascript
if (request.method === 'GET') {
  const cache = caches.default;
  const key = new Request(request.url, request);
  const hit = await cache.match(key);
  if (hit) {
    const hh = new Headers(hit.headers);
    hh.set('Access-Control-Allow-Origin','*');
    return new Response(hit.body, { status: hit.status, headers: hh });
  }
  const upstream = await fetch(upstreamUrl.toString(), { headers: request.headers });
  if (upstream.ok) await cache.put(key, upstream.clone());
  const hh = new Headers(upstream.headers);
  hh.set('Access-Control-Allow-Origin','*');
  return new Response(upstream.body, { status: upstream.status, headers: hh });
}
```

## Custom domain (optional)

wrangler.toml:
```toml
routes = [{ pattern = "api.yourdomain.com/*", custom_domain = true }]
```
Deploy again.

## Done

Use worker URL as base. All paths and queries forward unchanged. Anyone can use it unless you add restrictions. Consider whitelisting your origin.
