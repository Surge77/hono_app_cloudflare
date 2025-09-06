# Hono — Edge Web Framework (FPT Reference)

A concise, practical, first‑principles guide you can drop into any repo as `README_HONO.md`.

---

## 0) TL;DR (8ty)

* **What:** Hono is an **Express‑style web framework** for **edge runtimes** (Cloudflare Workers, Vercel Edge Functions, Deno, Bun, Fastly, Node).
* **Why:** You want **routing + middleware** with **web‑standard APIs** (`Request`/`Response`) and **tiny footprint**.
* **80/20:** Learn routing, middleware, context (`c`), adapters, and deployment. Avoid Node‑only libs (native addons, raw sockets).

---

## 1) First Principles (FPT)

**Problem:** Edge platforms are fast and global but **don’t run full Node.js** (no raw `net`, limited `fs`, no native addons). Classic Express apps break.

**Constraints:**

* Runtime = **V8 isolate** using **Fetch API** (web standards).
* No long‑lived servers; you return a `Response` to an incoming `Request`.
* Low latency, **stateless**, scale‑by‑default.

**Solution (Hono):**

* A lightweight **framework** offering Express‑like DX (routing, middleware) built **on web standards**, not Node APIs.
* Portable across edge providers; tiny, fast, TypeScript‑first.

**Result:** Familiar ergonomics + edge‑native performance.

---

## 2) Install & Hello World

```bash
# Pick your package manager
npm i hono   # or pnpm add hono / bun add hono
```

```ts
// src/app.ts
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hello Hono!'))
app.get('/user/:id', (c) => c.json({ id: c.req.param('id') }))

export default app
```

### Cloudflare Worker (Wrangler) entry

```ts
// src/worker.ts
import app from './app'
export default app
```

`wrangler.jsonc` (example)

```jsonc
{
  "name": "hono-app",
  "main": "src/worker.ts",
  "compatibility_date": "2025-09-06",
  "workers_dev": true,
  "nodejs_compat": true
}
```

Run:

```bash
npx wrangler dev     # local dev
npx wrangler publish # deploy
```

---

## 3) Core Concepts (the 20% to master)

* **Routing:** `app.get('/path', handler)`; path params via `c.req.param('id')`.
* **Context (`c`):** Hono’s request/response context: `c.req` (Request wrapper), `c.env` (bindings like KV/D1), `c.text/json/html()` helpers.
* **Middleware:** `app.use('*', middleware)` for logging, CORS, auth.
* **Adapters/Targets:** Runs on Cloudflare Workers, Bun, Deno, Vercel Edge, Fastly, Node.
* **Bindings:** Edge storage (KV, D1, R2, Durable Objects) available via `c.env` when provided by the platform.

---

## 4) Routing Cheatsheet

```ts
app.get('/health', (c) => c.text('ok'))
app.post('/items', async (c) => {
  const body = await c.req.json<{ name: string }>()
  return c.json({ id: crypto.randomUUID(), ...body }, 201)
})

app.get('/items/:id', (c) => {
  const { id } = c.req.param()
  return c.json({ id })
})

// Query strings
app.get('/search', (c) => {
  const q = c.req.query('q')
  return c.text(`q=${q}`)
})
```

---

## 5) Middleware Essentials

```ts
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'

app.use('*', logger())
app.use('/api/*', cors())

// Custom middleware
app.use('/secure/*', async (c, next) => {
  const token = c.req.header('authorization')?.replace('Bearer ', '')
  if (token !== c.env.API_TOKEN) return c.text('Unauthorized', 401)
  await next() // continue chain
})
```

---

## 6) Error Handling & Responses

```ts
app.onError((err, c) => {
  console.error(err)
  return c.json({ error: 'Internal Error' }, 500)
})

app.notFound((c) => c.json({ error: 'Not Found' }, 404))
```

Response helpers:

* `c.text(str, status?)`
* `c.json(obj, status?)`
* `c.html(html, status?)`
* `c.newResponse(body, init)` for full control.

---

## 7) Validation & Types

Use lightweight schema libs that work on the edge (no native addons):

```ts
import { z } from 'zod'

const Item = z.object({ name: z.string().min(1) })

app.post('/items', async (c) => {
  const parsed = Item.safeParse(await c.req.json())
  if (!parsed.success) return c.json({ errors: parsed.error.issues }, 400)
  return c.json(parsed.data, 201)
})
```

TypeScript tips:

* Infer types from Zod.
* Strongly type `c.env` via platform bindings (e.g., D1, KV) when your tooling supports it.

---

## 8) Auth Patterns (Edge‑friendly)

* **JWT (stateless):**

```ts
import { jwt } from 'hono/jwt'
app.use('/private/*', jwt({ secret: '🔑' }))
```

* **Session via Cookies:** Use `Set-Cookie` with `HttpOnly; Secure; SameSite=Strict`.
* Prefer **external IdPs** (Auth0, Clerk) or **OAuth over PKCE**; call via Fetch.

---

## 9) CORS, Headers, and Cache

```ts
import { cors } from 'hono/cors'
app.use('/api/*', cors({ origin: '*', allowHeaders: ['Authorization'] }))

app.get('/cache-me', (c) => {
  return c.text('ok', 200, { 'Cache-Control': 'max-age=60, s-maxage=300' })
})
```

* Use **`Cache-Control`** and platform caches (e.g., Cloudflare cache) for speed.

---

## 10) Data & Storage on the Edge

**Cloudflare** examples (available via `c.env` when bound in `wrangler.jsonc`):

* **KV** (key/value): config, feature flags, counters.
* **D1** (SQLite at edge): relational queries.
* **R2** (object storage): uploads, assets.
* **Durable Objects**: stateful coordination, rooms/locks.

Example (D1):

```ts
app.get('/todos', async (c) => {
  const rs = await c.env.DB.prepare('SELECT id, title FROM todos').all()
  return c.json(rs.results)
})
```

---

## 11) Deployment Targets

* **Cloudflare Workers / Pages** (Wrangler) — most common.
* **Vercel Edge Functions** — adapt export.
* **Deno Deploy / Bun** — native Fetch runtime.
* **Fastly Compute\@Edge** — supported via adapter.
* **Node** — supported, but edge gains are in edge runtimes.

> Rule of thumb: pick the platform where your data lives or where most users are.

---

## 12) When to Use vs. Avoid

**Use Hono when:**

* You need **global low‑latency APIs**.
* You can use **web‑standard libraries** (no native addons).
* You want **small, maintainable** HTTP services with middleware.

**Avoid / reconsider when:**

* You depend on **Node‑only features** (`fs`, raw sockets, native modules like `sharp` w/out edge build).
* You need **long‑lived connections** beyond platform allowances.
* Your DB driver requires TCP sockets (prefer HTTP‑based access or Cloudflare D1/KV/R2).

---

## 13) Express → Hono Migration (mental map)

* `req.params.id` → `c.req.param('id')`
* `req.query.q` → `c.req.query('q')`
* `res.json(x)` → `c.json(x)`; `res.send()` → `c.text()`/`c.html()`
* Middleware pattern stays: `(c, next) => { ...; await next() }`
* No `app.listen()`; **export default app** for the runtime.

---

## 14) Performance & Footprint

* Hono is **tiny (\~tens of KB)** and extremely **fast**.
* Cold starts are negligible on edge isolates; concurrency is high by design.
* Use streaming responses (`return c.body(stream)`) for large payloads.

---

## 15) Project Template (Cloudflare Workers)

```
my-hono-app/
├─ src/
│  ├─ app.ts        # routes & middleware
│  └─ worker.ts     # export default app
├─ wrangler.jsonc   # runtime config & bindings
├─ package.json
└─ README_HONO.md
```

`package.json` (scripts)

```json
{
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler publish",
    "typecheck": "tsc --noEmit"
  }
}
```

---

## 16) Common Misconceptions (cm)

1. *“Hono replaces Next.js.”* → No. Hono = backend/API routing; Next = full React framework.
2. *“Edge == Node in the cloud.”* → No. Edge uses web standards; avoid Node‑specific APIs.
3. *“I can lift‑and‑shift an Express app unchanged.”* → Usually not; refactor to Fetch‑style handlers.
4. *“No DB on edge.”* → Use D1/KV/R2 or HTTP DB proxies; avoid raw TCP drivers.

---

## 17) Security Notes

* Always set strict cookies: `HttpOnly; Secure; SameSite=Strict`.
* Validate all inputs (Zod/Valibot) and sanitize outputs.
* Use per‑request secrets from bindings / environment (avoid bundling secrets).
* Enable CORS precisely; avoid `*` for credentialed requests.

---

## 18) Troubleshooting

* **“Module not found: fs/net/crypto native”** → Replace with edge‑compatible libs or enable `nodejs_compat` (still limited).
* **Binary deps (e.g., sharp, bcrypt)** → Use edge builds or alternatives (e.g., S3‑style image services, Web Crypto, Argon2 WASM).
* **Routes not hitting** → Check `wrangler.jsonc` `main`, `routes`, `workers_dev` flags.

---

## 19) Mini Cookbook

**JWT Auth**

```ts
import { jwt } from 'hono/jwt'
app.use('/private/*', jwt({ secret: c => c.env.JWT_SECRET }))
```

**CORS**

```ts
import { cors } from 'hono/cors'
app.use('/api/*', cors({ origin: ['https://app.example.com'] }))
```

**Basic Rate Limit (KV)**

```ts
app.use('*', async (c, next) => {
  const ip = c.req.header('cf-connecting-ip') ?? 'anon'
  const key = `rate:${ip}`
  const hits = Number(await c.env.KV.get(key)) || 0
  if (hits > 100) return c.text('Too Many Requests', 429)
  await c.env.KV.put(key, String(hits + 1), { expirationTtl: 60 })
  await next()
})
```

---

## 20) Socratic Q\&A (compact, FPT)

1. **Q:** What breaks when moving Express to edge? **A:** Node‑only APIs and native addons.
2. **Q:** Why does edge reduce latency? **A:** Code runs near users on CDN POPs.
3. **Q:** Why Hono over raw Fetch handlers? **A:** Routing/middleware DX, structure, reuse.
4. **Q:** When is `nodejs_compat` enough? **A:** For light Node shims; not for sockets/native.
5. **Q:** Where do I keep state? **A:** KV/D1/R2/DO; avoid in‑memory globals.
6. **Q:** How to secure endpoints? **A:** JWT/cookies, validate inputs, least‑privilege bindings.
7. **Q:** What’s the minimal deploy step? **A:** `wrangler publish` with correct `main`.
8. **Q:** How do I log? **A:** `console.log`, analytics engine, or DO aggregator.
9. **Q:** How to stream responses? **A:** Use `ReadableStream`/`c.body(stream)`.
10. **Q:** How to handle errors globally? **A:** `app.onError`, `app.notFound`.
11. **Q:** Can I SSR React with Hono? **A:** Yes, but for full app use Next/Remix; Hono excels at APIs.
12. **Q:** How to call external APIs? **A:** `fetch()` with proper timeouts/retries.
13. **Q:** How to cache API responses? **A:** `Cache-Control` + platform cache APIs.
14. **Q:** Migrate DB from TCP driver? **A:** Use HTTP proxy, Prisma Data Proxy, or D1.
15. **Q:** Versioning APIs? **A:** Namespace routes (`/v1`, `/v2`) or headers.
16. **Q:** Testing? **A:** Super‑fetch style tests using standard `Request`/`Response` in Node/Bun.
17. **Q:** Observability? **A:** Add request IDs, structured JSON logs, edge analytics.

---

## 21) Copy‑Paste Templates

**`wrangler.jsonc`**

```jsonc
{
  "name": "hono-app",
  "main": "src/worker.ts",
  "compatibility_date": "2025-09-06",
  "workers_dev": true,
  "nodejs_compat": true,
  "vars": { "API_TOKEN": "changeme" },
  "kv_namespaces": [
    { "binding": "KV", "id": "<kv-id>" }
  ],
  "d1_databases": [
    { "binding": "DB", "database_id": "<d1-id>", "database_name": "app-db" }
  ]
}
```

**`src/app.ts`**

```ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { logger } from 'hono/logger'

const app = new Hono<{ Bindings: { KV: KVNamespace; DB: D1Database; API_TOKEN: string } }>()

app.use('*', logger())
app.use('/api/*', cors())

app.get('/', (c) => c.text('ok'))

app.get('/todos', async (c) => {
  const rs = await c.env.DB.prepare('SELECT id, title FROM todos').all()
  return c.json(rs.results)
})

export default app
```

**`src/worker.ts`**

```ts
import app from './app'
export default app
```

---

## 22) Checklist Before Prod

* [ ] Replace any Node‑only packages.
* [ ] Add global error handler & 404.
* [ ] Configure CORS and auth.
* [ ] Add caching headers where safe.
* [ ] Bind KV/D1/R2 and type `c.env`.
* [ ] Load test your hot paths.
* [ ] Observability: logs, metrics, alerts.

---

## 23) FAQ

* **Is Hono a library or framework?** Both—an npm package that provides a web framework.
* **Can I run it on Node?** Yes, but edge runtimes unlock the big wins.
* **Does it replace Next.js?** No—pair Hono APIs with a frontend framework, or use Next for full‑stack.

---

**Keep this file around as your edge API playbook.**
