# 11 · Failure Modes & Edge Cases

> **What you'll be able to answer after this chapter**
> - What can go wrong at each layer, and what status/error results? (Failure)
> - How Express behaves on empty, malformed, boundary, and concurrent inputs. (Edge cases)
> - What's fragile or surprising, and how to avoid the footguns. (Gotchas)

This is the consolidated "what happens if…" reference. Every row is grounded in the tests
cited elsewhere; use it as a lookup table. HTTP status codes below are what the client
actually receives.

---

## 1. Status-code outcomes at a glance

| Situation | Result | Source |
|---|---|---|
| No route matches, stack exhausted | **404** (finalhandler) | `test/app.router.js:80-82` |
| Path exists but method not registered | **404** (no auto-405) | `test/app.options.js:52-60` |
| Handler throws / `next(err)` / rejected promise, no error mw | **500** (finalhandler, logs unless test) | `test/app.routes.error.js:9-23` |
| Automatic OPTIONS on a known path | **200** + `Allow` header | `test/app.options.js:7-18` |
| Conditional GET, validators match | **304** (via `res.send`/static) | `test/res.send.js:315-344` |
| `res.format` no match, no `default` | **406** (`createError`) | `test/res.format.js:239-247` |
| Body over `limit` | **413** | `test/express.json.js:121-194` |
| Unsupported charset / encoding | **415** | `test/express.json.js:641,693` |
| Invalid JSON / bad Content-Length | **400** | `test/express.json.js:44-71` |
| `verify` hook throws | **403** | `test/express.json.js:399-411` |
| Static traversal past root, `fallthrough:false` | **403** `ForbiddenError` | `test/express.static.js:575-590` |
| Static traversal past root, **default** `fallthrough:true` | **404** (silent) | `test/express.static.js:271-275` |
| Static file missing (fallthrough default) | **404** | `test/express.static.js:248-252` |
| `urlencoded` over `parameterLimit` | **413** `[parameters.too.many]` | `test/express.urlencoded.js:328` |

## 2. Throws that become 500

Express methods that **throw** (the router converts throws to `next(err)`, so they surface as
**500** unless an error handler intervenes):

| Call | Throws | Where |
|---|---|---|
| `res.status(x)` with non-integer | `TypeError` "must be an integer" | `lib/response.js:67-68` |
| `res.status(x)` outside 100–999 | `RangeError` | `lib/response.js:71-72` |
| `req.get()` / `req.get(42)` | `TypeError` (name required / must be string) | `lib/request.js:65-71` |
| `res.set('Content-Type', [array])` | `TypeError` "cannot be set to an Array" | `lib/response.js:677` |
| `res.vary()` (no field) | throws `/field.*required/` (from `vary`) | `test/res.vary.js:8-21` |
| `res.cookie(n,v,{signed:true})` without secret | `Error` "cookieParser secret required" | `lib/response.js:750-752` |
| `res.cookie(n,v,{maxAge:'foo'})` | `/option maxAge is invalid/` (from `cookie`) | `test/res.cookie.js:174-185` |
| `res.sendFile(relativePath)` without `root` | `TypeError` "must be absolute or specify root" | `lib/response.js:394-396` |
| `app.use()` with no fn | `TypeError` "requires a middleware function" | `lib/application.js:212-214` |
| `app.engine(ext, non-fn)` | `Error` "callback function required" | `lib/application.js:295-297` |

Throws at **configuration time** (fail fast, not per-request): `app.set('etag'|'query parser',
<unknown>)` → `TypeError: unknown value for … function` (`lib/utils.js:148,180`).

## 3. Empty / missing input

| Input | Behavior |
|---|---|
| `req.get('X')` for absent header | `undefined` (`lib/request.js:81`) |
| `req.query` with parsing disabled (`query parser:false`) | `Object.create(null)` (empty, null-proto) |
| `req.range(size)` with no `Range` header | `undefined` (`lib/request.js:215-216`) |
| `req.is('json')` with no Content-Type | `false` (`test/req.is.js:51-64`) |
| `req.host`/`req.hostname` with no host header | `undefined` (`lib/request.js:430,447`) |
| `req.subdomains` with no host | `[]` (`lib/request.js:384-386`) |
| `req.accepts('json')` with no `Accept` header | truthy (everything acceptable) |
| `res.send()` / `res.send(undefined)` | no Content-Length, no ETag, empty response |
| `res.send(null)` | explicit `Content-Length: 0`, empty body |
| `res.json(undefined)` with `json escape` on | empty body, no crash (`test/res.json.js:127-140`) |
| Body parser, empty body **with matching type** | parses; `req.body` = `{}` / `""` / empty Buffer (`test/express.json.js:19-41`) |
| Body parser **skips** (no/wrong content-type) | `req.body` left **undefined** (not `{}`; `test/express.json.js:311-317`, `History.md` v5) |
| `router.handle` with `url:''` or no url | no middleware runs; `done` called (`test/Router.js:44-62`) |
| Empty `Route` (no handlers) | **404** (`test/app.route.js:55-63`) |

## 4. Malformed input

| Input | Behavior |
|---|---|
| Malformed percent-encoding in path (`/%foobar`) | **400**, params not populated (`test/app.router.js:107-117`) |
| Malformed URL to static (`/%`), `fallthrough:true` | **404**; `fallthrough:false` → **400** `BadRequestError` |
| Malformed gzip body | **400** (`test/express.json.js:701-707`) |
| Invalid JSON primitive under `strict` | **400** `[entity.parse.failed]` |
| Syntactically invalid `Range` header | **200** full content (static) / handled by range-parser (`req.range` → `-2`) |
| Non-string redirect/status args | `depd` deprecation warnings (non-fatal), best-effort behavior (`lib/response.js:826-836`) |
| Unparseable existing Content-Type + `res.send(string)` | does **not** throw; charset appended (`test/res.send.js:138-149`) |

## 5. Boundary conditions

- **Status range:** 100–999 accepted; 99 and 1000 throw. Non-standard codes (700–900) are
  allowed (`test/res.status.js`).
- **Content-Length fast path:** strings <1000 bytes with no ETag use `Buffer.byteLength`
  instead of allocating (`lib/response.js:172-174`).
- **`maxAge` clamps:** static/`sendFile` `maxAge: Infinity` → 1 year (`31536000`)
  (`test/express.static.js:460-465`).
- **`parameterLimit` floor comparison:** `10.1` behaves like `10`
  (`test/express.urlencoded.js:345-351`).
- **Splat matching:** `*name` matches **1+** segments, not zero; wrap optional `{/*name}` to
  allow zero (`test/app.router.js:767-777`).
- **6000-deep stacks:** no stack overflow — dispatch is iterative/deferred
  (`test/Router.js:91-160`).
- **`subdomain offset: 0`:** returns all reversed host labels; IP hosts stay a single element
  (`test/req.subdomains.js:96-137`).
- **205 vs 204/304:** 205 keeps Content-Type but forces `Content-Length: 0`; 204/304 strip
  Content-Type/Length/Transfer-Encoding (`lib/response.js:197-209`).

## 6. Concurrency, ordering & idempotency

- **Per-request isolation:** all `req`/`res` getters are pure over per-request state; async
  params across parallel requests don't cross-contaminate (`test/Router.js:599-635`).
- **Middleware order = registration order**, across a single unified stack of interleaved
  middleware and routes (`test/app.router.js:1115-1152`).
- **`res.locals` is per-request** (created in `app.handle`, `lib/application.js:173-175`);
  **`app.locals` persists across requests** — don't stash per-request data in `app.locals`.
- **Param callbacks fire once per (request, value)** (`test/app.param.js:60-114`).
- **Duplicate body parsers are idempotent** — the second parse short-circuits on `req.body`
  already set.
- **`sendfile` double-callback guard:** a single `done` flag prevents the file-stream state
  machine from calling back twice across abort/error/finish races (`lib/response.js:925`).
- **Throw after `res.end()` must not corrupt the sent response** — regression-guarded
  (`test/regression.js:6-20`: `res.end('yay')` then `throw` still yields `200 'yay'`).
- **Responding twice / writing after the response is sent** is **not** something Express guards
  against — `res` is `Object.create(http.ServerResponse.prototype)` (`lib/response.js:43`), so a
  second `res.send`/`res.json`/`res.set`/`res.redirect` after headers are flushed hits Node's
  native `ERR_HTTP_HEADERS_SENT` ("Cannot set headers after they are sent to the client"), and a
  body write after `end()` throws `ERR_STREAM_WRITE_AFTER_END`. Common triggers: forgetting to
  `return` after `res.send(...)` and then falling through to another response, or calling
  `next()` after already responding so a later layer responds again. If that error is thrown
  synchronously inside a handler, the router converts it to `next(err)` (→ your error middleware
  or finalhandler's 500) — but if the headers are already sent, `res.headersSent` is true and the
  error handler can't change the response, only log it. **Guard:** always `return` after
  responding, and check `res.headersSent` before writing in late/async callbacks.
- **Resolved promises do NOT auto-call `next`** — an `async` handler that only returns a
  resolved promise, without responding or calling `next()`, hangs the request
  (`test/app.router.js:1004-1022`). This is a real footgun for `async` handlers.
- **View cache races:** concurrent cache misses may each build a `View` (no dedupe); results
  converge, work is duplicated (inference, `lib/application.js:544-570`).
- **AsyncLocalStorage context survives body parsing and file sends** — a protected invariant
  across all parsers and `res.sendFile`.

## 7. Client-abort & network failures

- **`res.sendFile`/`res.download` client abort** → callback gets `err.code === 'ECONNABORTED'`
  (or `ECONNRESET` treated as abort); no crash, error is swallowed if no callback
  (`lib/response.js:411-413,968-983`; `test/res.sendFile.js:147-173`).
- **Write errors** during file send (`err.syscall === 'write'`) are not passed to `next`
  (`lib/response.js:411-413`) — the connection is already gone.
- **Directory requested via `sendFile`** → `EISDIR` → `next()` (falls through to 404).

## 8. The v4→v5 breaking-change traps

These are the migration footguns (from `History.md`); code written for v4 breaks on v5:

| v4 pattern | v5 behavior | Fix |
|---|---|---|
| `res.send(status, body)` / `res.json(status, obj)` | overload removed | `res.status(s).send(body)` |
| `res.redirect(url, status)` | arg order removed | `res.redirect(status, url)` |
| `res.redirect('back')` / `res.location('back')` | no magic string | `req.get('Referrer') || '/'` |
| `res.status('200')` (string) | throws `TypeError` | pass an integer |
| Route `'/user/:id?'` (optional) | v8 syntax | `'/user/:id{/:id}'`-style `{...}` braces |
| Route `'/files/:file*'` (splat) | v8 syntax | `'/files/*file'` (yields an **array**) |
| `req.query` nested by default | default `'simple'` (flat) | `app.set('query parser','extended')` |
| `urlencoded()` nested by default | default `extended:false` | `express.urlencoded({ extended:true })` |
| `req.body` always `{}` | not always initialized | check before reading |
| `res.clearCookie(n, { maxAge })` | maxAge/expires ignored | rely on the forced past-expiry |

## 9. Silent / surprising behaviors (footguns)

- **`app.get('name')` with one arg reads a setting**, not a route. Forgetting the handler
  silently registers nothing (`lib/application.js:473-475`).
- **`res.send(object)` becomes JSON** — numbers/booleans/arrays/objects all go to `res.json`,
  not text (`lib/response.js:156`).
- **`res.location('back')` emits `Location: back`** literally — dead JSDoc (`lib/response.js:783-785`).
- **`_locals` passed to `res.render` is dropped** (overwritten by `res.locals`)
  (`lib/response.js:911`).
- **`case sensitive routing`/`strict routing` set after the first route have no effect**
  (router memoized, `lib/application.js:69-82`).
- **`views` default is CWD-relative**, fixed at `express()` time (`lib/application.js:135`).
- **`extended:true` query/urlencoded opens `__proto__` keys** (`allowPrototypes:true`,
  `lib/utils.js:269`) — avoid on untrusted input.
- **`X-Powered-By: Express` leaks by default** — disable for hardening.
- **`app.all` middleware runs on OPTIONS but is excluded from `Allow`**
  (`test/app.options.js:34-50`).
- **`res.set` after `res.append` resets** the accumulated multi-value header
  (`test/res.append.js:44-63`).
- **Nested routers overwrite `req.params`** unless created with `mergeParams:true`.

## 10. What is NOT Express's responsibility

Common expectations that Express deliberately does **not** handle (you must add middleware):
- **CORS, CSRF, rate limiting, sessions, authentication** — all external middleware
  (`express-session`, `cookie-parser`, etc.; see `examples/auth`, `examples/session`).
- **Automatic 405 Method Not Allowed** — you get 404; implement 405 yourself if needed.
- **Request logging** — use `morgan` (a devDependency used in examples).
- **HTTPS/TLS** — pass the app to `https.createServer(opts, app)` yourself.
- **Clustering, graceful shutdown, timeouts** — Node/`http.Server` concerns.
- **`req.body`** exists only if you add a body parser; **`req.cookies`** only with
  `cookie-parser`.

## Where to look

- Every subsystem chapter's "Errors, edge cases & gotchas" section for the detail behind these
  rows.
- `test/regression.js` — the explicitly-protected non-regression behaviors.
- `History.md` — the authoritative list of v4→v5 breaking changes.

## Open questions

- Precedence when multiple parser errors apply simultaneously (bad Content-Length **and** bad
  charset) is untested (`body-parser` internal).
- `req.is` `null` and `req.range` `-1`/`-2` returns are documented but not exercised in-repo.

**Next:** [12 · How Do I…? (Cookbook)](12-how-do-i.md).
