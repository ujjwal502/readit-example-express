# 13 · Glossary

Domain and codebase-specific terms, defined as they mean **in this repo**. Each links to where
it's defined or used.

---

**app / application** — The object returned by `express()`. Simultaneously a request-listener
function `(req,res,next)`, an EventEmitter, and a settings/methods bag
(`lib/express.js:36-56`). See [Chapter 3](03-the-application-object.md).

**`app.handle`** — The per-request entry point. Sets `X-Powered-By`, links `req`/`res`, swaps
their prototypes, creates `res.locals`, and dispatches to the router
(`lib/application.js:152-178`).

**arity-4 middleware** — A middleware declared with four parameters `(err, req, res, next)`. Its
arity is the *only* thing that marks it as an **error handler**; the router routes active errors
only to these ([Chapter 4](04-routing-and-middleware.md#error-handling-middleware--arity-4)).

**base router** — The single `Router` instance each app lazily builds on first access, using the
`case sensitive routing`/`strict routing` settings (`lib/application.js:69-82`).

**body parser** — Middleware (`express.json/urlencoded/text/raw`) that reads the request body
into `req.body`. Re-exported from `body-parser` (`lib/express.js:77-81`). See
[Chapter 8](08-bundled-middleware.md).

**compiled setting / `… fn`** — A setting value turned into a fast function at set time. `etag`
→ `etag fn`, `query parser` → `query parser fn`, `trust proxy` → `trust proxy fn`
(`lib/application.js:363-380`, `lib/utils.js`).

**decoration / prototype swap** — Express's core technique: per request, `app.handle` runs
`Object.setPrototypeOf(req, this.request)` / `(res, this.response)` to insert Express's API
between the native object and Node's prototype, with zero allocation
(`lib/application.js:169-170`).

**engine** — A template-rendering function with signature `(path, options, callback)` registered
via `app.engine(ext, fn)` (`lib/application.js:294-308`). Express ships none of its own. See
[Chapter 7](07-views-and-rendering.md).

**`finalhandler`** — The external package used as the end-of-stack `done` callback; produces the
default **404** (empty stack) or **500** (unhandled error) and logs via `onerror`
(`lib/application.js:154-157`).

**layer** — The internal pairing of a compiled path matcher with a handler, inside the `router`
package. Not defined in this repo; behavior observed via tests.

**locals** — Template variables. **`app.locals`** is app-wide and persists across requests
(`lib/application.js:125`); **`res.locals`** is per-request (`lib/application.js:173-175`).
Precedence in rendering: `app.locals` < `res.locals`/`_locals` < render-call options.

**middleware** — A function `(req, res, next)` run by the router in registration order. The
atomic unit of an Express app. See [Chapter 4](04-routing-and-middleware.md).

**mountpath / parent** — Set when an app/router is mounted via `app.use(path, subApp)`:
`subApp.mountpath = path`, `subApp.parent = app` (`lib/application.js:226-227`). `app.path()`
walks them for the absolute path.

**`mount` event** — Emitted on a sub-app when it's mounted; its handler links the child's
`request`/`response`/`settings`/`engines` prototypes to the parent's
(`lib/application.js:109-122`, `:240`).

**`next()`** — The function that advances dispatch. `next()` → next layer; `next('route')` → skip
rest of the current Route; `next('router')` → exit the current router; `next(err)` → jump to
error handling ([Chapter 4](04-routing-and-middleware.md#the-four-next-behaviors)).

**param callback** — A function registered with `app.param(name, fn)` that preprocesses a route
parameter, firing once per request per value (`lib/application.js:322-334`;
[Chapter 4 §5](04-routing-and-middleware.md#5-appparamname-fn--parameter-preprocessing)).

**`req` (request)** — Node's `http.IncomingMessage` with Express's `lib/request.js` prototype
layered on. Read-side helpers. See [Chapter 5](05-the-request-object.md).

**`req.body` / `req.params` / `req.query`** — Parsed request body (from a body parser), decoded
route parameters (from the router), and parsed query string (via `query parser fn`),
respectively.

**`res` (response)** — Node's `http.ServerResponse` with Express's `lib/response.js` prototype
layered on. Write-side helpers. See [Chapter 6](06-the-response-object.md).

**Rosetta Flash mitigation** — The `/**/` prefix `res.jsonp` prepends to its body to neutralize a
content-sniffing Flash attack (`lib/response.js:300-302`).

**Route** — An isolated middleware stack for one path, with per-method handlers; returned by
`app.route(path)` or `express.Route`. `app.get('/x', fn)` is sugar for
`app.route('/x').get(fn)` (`lib/application.js:256-258,478-479`).

**Router** — The dispatch engine: an ordered stack of layers with path matching and `next`
semantics. In Express 5 it's the external `router` package; `express.Router === require('router')`
(`lib/express.js:19,71`).

**settings** — The per-app configuration bag `app.settings` (null-prototype). Read/write via
`app.set/get/enable/disable` (`lib/application.js:351-465`). Full catalog:
[Chapter 3](03-the-application-object.md#the-complete-settings-catalog).

**splat / wildcard** — An Express 5 `*name` route segment that captures **an array** of path
segments; matches 1+ segments (`test/app.router.js:704-777`).

**strict routing** — Setting that makes trailing slashes significant (`/foo` ≠ `/foo/`). Read once
at router construction (`lib/application.js:76`).

**trust proxy** — Setting that controls whether `X-Forwarded-*` headers are trusted for
`req.ip/ips/protocol/host/hostname`. Default `false`. Compiled by `compileTrust`
(`lib/utils.js:194-214`). See [Chapter 5 §7](05-the-request-object.md#7-trust-proxydependent-properties).

**View** — The `lib/view.js` object that resolves a template name to a file and invokes the
engine. Backs `res.render`/`app.render`. See [Chapter 7](07-views-and-rendering.md).

**view cache** — Setting (`view cache`) that caches `View` objects by render name; on in
production, off in development by default (`lib/application.js:138-140,544-570`).

**`x-powered-by`** — Setting (default enabled) that emits `X-Powered-By: Express`; disable to hide
the header (`lib/application.js:94,160-162`).

---

**Next:** back to the [Table of Contents](README.md).
