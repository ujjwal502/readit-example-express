# 03 · The Application Object

> **What you'll be able to answer after this chapter**
> - What exactly does `express()` construct, and what state does an app hold? (Architecture / State)
> - Every setting Express supports: default, allowed values, and what it changes. (Interfaces / Config)
> - How settings are compiled, how `app.get` is overloaded, and how mounting/inheritance works. (Mechanism)
> - How `app.listen`, `app.render`, and per-request decoration behave, with edge cases. (Control flow / Failure)

**Source of truth:** `lib/application.js` (631 lines) and `lib/express.js` (the factory).
Grounded throughout by `test/config.js`, `test/app.js`, `test/app.listen.js`,
`test/app.locals.js`, `test/app.engine.js`, `test/app.render.js`, `test/app.request.js`,
`test/app.response.js`, `test/app.head.js`, `test/app.options.js`, `test/exports.js`.

---

## 1. What `express()` builds

`express()` is `createApplication()` (`lib/express.js:36-56`). It returns a **function** and
decorates it:

```js
// lib/express.js:36-55
function createApplication() {
  var app = function(req, res, next) {   // ← the app is a request listener
    app.handle(req, res, next);
  };

  mixin(app, EventEmitter.prototype, false);   // → app.on/emit  (the 'mount' event)
  mixin(app, proto, false);                     // → app.use/get/set/listen/... from application.js

  // per-app req/res prototypes, each carrying an own `app` back-reference
  app.request  = Object.create(req, { app: { configurable:true, enumerable:true, writable:true, value: app } })
  app.response = Object.create(res, { app: { configurable:true, enumerable:true, writable:true, value: app } })

  app.init();
  return app;
}
```

- `mixin` is `merge-descriptors` — it copies property *descriptors* (getters/setters
  included), with `false` meaning "don't overwrite existing props."
- `app.request`/`app.response` are per-app objects whose prototype is the shared
  `lib/request.js`/`lib/response.js` exports, and which carry an own `app` property. That
  `app` back-reference is how `req.app`/`res.app` work, and why every `req`/`res` can reach
  its application's settings (`this.app.get(...)`).

`app.init()` (`lib/application.js:59-83`) sets up the mutable state:

```js
// lib/application.js:62-82
this.cache    = Object.create(null);   // view cache: name → View
this.engines  = Object.create(null);   // '.ext' → engine fn
this.settings = Object.create(null);   // setting name → value
this.defaultConfiguration();
// lazy base router, built on first access:
Object.defineProperty(this, 'router', { configurable:true, enumerable:true,
  get: function getrouter() {
    if (router === null) {
      router = new Router({
        caseSensitive: this.enabled('case sensitive routing'),
        strict: this.enabled('strict routing')
      });
    }
    return router;
  }});
```

**The state an app holds:** `settings` (config bag), `engines` (template registry),
`cache` (view cache), `locals` (app-wide template vars), `mountpath`/`parent` (mounting),
`request`/`response` (per-app prototypes), and a lazily-built `router`. All three of
`cache`/`engines`/`settings` are **null-prototype objects**, deliberately: it lets you use
keys like `hasOwnProperty` as setting names and prevents prototype-pollution via a setting
name (`test/config.js:14-18,71-74`).

## 2. Default configuration

`defaultConfiguration()` (`lib/application.js:90-141`) runs during `init`, establishing
every default. In order:

```js
var env = process.env.NODE_ENV || 'development';   // :91
this.enable('x-powered-by');                        // :94  → adds X-Powered-By: Express
this.set('etag', 'weak');                           // :95  → compiles 'etag fn' (weak)
this.set('env', env);                               // :96
this.set('query parser', 'simple');                 // :97  → 'query parser fn' = querystring.parse
this.set('subdomain offset', 2);                    // :98
this.set('trust proxy', false);                     // :99  → 'trust proxy fn' (trust nothing)
// ... trust-proxy inheritance symbol (see §6) ...
this.locals = Object.create(null);                  // :125
this.mountpath = '/';                               // :128
this.locals.settings = this.settings;               // :131  (same reference)
this.set('view', View);                             // :134  (lib/view.js constructor)
this.set('views', resolve('views'));                // :135  → <cwd>/views
this.set('jsonp callback name', 'callback');        // :136
if (env === 'production') this.enable('view cache');// :138-140
```

Two consequences worth remembering:
- **`env` defaults to `'development'`** when `NODE_ENV` is empty or unset
  (`test/app.js:106-120`). In production, the view cache is on by default; in development,
  off — so dev re-stats template files every render (`test/app.render.js:230-288`).
- **`views` is resolved against `process.cwd()` at app-creation time** (`:135`), not the
  module directory — a common deployment gotcha.

## 3. Per-request decoration: `app.handle`

Every request enters here (the app-as-function calls it, `lib/express.js:37-39`):

```js
// lib/application.js:152-178
app.handle = function handle(req, res, callback) {
  var done = callback || finalhandler(req, res, {           // :154 default end-of-stack
    env: this.get('env'),
    onerror: logerror.bind(this)
  });
  if (this.enabled('x-powered-by')) res.setHeader('X-Powered-By', 'Express');  // :160-162
  req.res = res; res.req = req;                              // :165-166 circular refs
  Object.setPrototypeOf(req, this.request)                  // :169 decorate req
  Object.setPrototypeOf(res, this.response)                 // :170 decorate res
  if (!res.locals) res.locals = Object.create(null);        // :173-175 per-request locals
  this.router.handle(req, res, done);                       // :177 dispatch
};
```

- If you pass your own `callback` (as when mounting a sub-app), it is used instead of
  `finalhandler`; otherwise `finalhandler` produces the default **404** (empty stack) or
  **500** (unhandled error) response and logs via `logerror`.
- `logerror` calls `console.error(err)` **unless** `env === 'test'`
  (`lib/application.js:615-618`) — which is why the test suite (with `NODE_ENV=test`, set by
  `test/support/env.js`) stays quiet even while exercising error paths.

## 4. The settings API

Four methods, all thin wrappers over `this.settings` (`lib/application.js:351-465`):

```js
app.set(name, val)   // store + compile side-effects; 1-arg form is a GETTER
app.get(name)        // 1-arg → app.set(name) getter; 2+ args → register a GET ROUTE (!)
app.enable(name)     // set(name, true)
app.disable(name)    // set(name, false)
app.enabled(name)    // Boolean(set(name))
app.disabled(name)   // !set(name)
```

`app.set` with one argument returns the value (`:352-355`). With two, it stores and returns
`this` for chaining — even if `val` is `undefined` (`test/config.js:25-28`). Then it runs a
`switch` that **compiles** three special settings into companion `… fn` keys:

```js
// lib/application.js:363-380
switch (setting) {
  case 'etag':          this.set('etag fn', compileETag(val)); break;
  case 'query parser':  this.set('query parser fn', compileQueryParser(val)); break;
  case 'trust proxy':
    this.set('trust proxy fn', compileTrust(val));
    Object.defineProperty(this.settings, trustProxyDefaultSymbol, { configurable:true, value:false });
    break;
}
```

The compilers live in `lib/utils.js` (covered in [Chapter 9](09-cross-cutting-concerns.md)).
Key behaviors: `compileETag`/`compileQueryParser` **throw** on an unknown string value
(`TypeError: unknown value for etag function: …`, `lib/utils.js:148`,`:180`) — so a typo
like `app.set('etag', 'wek')` fails loudly at set time, not at request time
(`test/config.js:41-45`). Passing a function is always accepted verbatim and becomes the
`… fn` directly (`test/config.js:47-52,56-61`).

### The `app.get` overload (a famous footgun)

`app.get` is defined inside the verb-registration loop, not as a normal method
(`lib/application.js:471-482`):

```js
methods.forEach(function (method) {
  app[method] = function (path) {
    if (method === 'get' && arguments.length === 1) {
      return this.set(path);          // ← app.get('title') is a SETTINGS GETTER
    }
    var route = this.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

So `app.get('view engine')` **reads a setting**, while `app.get('/users', handler)`
**registers a GET route**. The disambiguation is purely `arguments.length === 1`. Every
other verb (`post`, `put`, `delete`, …) is a pure route registrar; only `get` is
overloaded, for historical reasons.

### The complete settings catalog

Every setting Express reads, its default, allowed values, and effect — grounded in
`lib/application.js`, `lib/utils.js`, and the consuming code:

| Setting | Default | Allowed values | Effect / consumed at |
|---|---|---|---|
| `env` | `NODE_ENV` or `'development'` | any string | `finalhandler` env + error-log gate (`:154`, `:617`). |
| `x-powered-by` | enabled (`true`) | boolean | Adds `X-Powered-By: Express` (`:160-162`). Disable to hide the header. |
| `etag` | `'weak'` | `'weak'`/`true`, `'strong'`, `false`, or a `fn(body,enc)` | Compiles `etag fn`; used by `res.send` (`lib/response.js:162`). Unknown string throws. |
| `etag fn` | derived | (internal) | The compiled ETag generator. |
| `query parser` | `'simple'` | `'simple'`/`true`, `'extended'`, `false`, or a `fn(str)` | Compiles `query parser fn`; used by `req.query` (`lib/request.js:231`). Unknown string throws. |
| `query parser fn` | derived | (internal) | Compiled parser (`querystring.parse` / `qs.parse` / custom / off). |
| `subdomain offset` | `2` | integer | How many trailing host labels are "the domain"; used by `req.subdomains` (`lib/request.js:388`). |
| `trust proxy` | `false` | boolean, hop-count number, comma-string, IP/subnet array, or `fn(addr,i)` | Compiles `trust proxy fn`; governs all `X-Forwarded-*` trust (`req.ip/ips/protocol/host/hostname`). |
| `trust proxy fn` | derived | (internal) | Compiled trust predicate. |
| `view` | `View` (`lib/view.js`) | a constructor `(name,opts)` with `.path` + `.render` | The view class used by `app.render` (`:552`). Override to change view resolution. |
| `views` | `resolve('views')` (cwd) | string or array of dirs | Template lookup root(s) (`lib/view.js:104-123`). |
| `view cache` | on in production, else off | boolean | Caches `View` objects by name (`:539-570`). |
| `view engine` | unset | extension string | Default engine when a render name has no extension (`:553`). |
| `jsonp callback name` | `'callback'` | string | Query key holding the JSONP callback (`lib/response.js:269`). |
| `json escape` | unset | boolean | Escapes `<`,`>`,`&` in `res.json`/`jsonp` output (`lib/response.js:237,265`). |
| `json replacer` | unset | `JSON.stringify` replacer | Passed to `JSON.stringify` in `res.json`/`jsonp`. |
| `json spaces` | unset | number/string | Pretty-print indent for `res.json`/`jsonp`. |
| `case sensitive routing` | off | boolean | `/Foo` ≠ `/foo`. Read **once** at router construction (`:75`). |
| `strict routing` | off | boolean | `/foo` ≠ `/foo/`. Read **once** at router construction (`:76`). |

> **Gotcha:** `case sensitive routing` and `strict routing` are read a single time, when the
> router is first accessed (`:69-82`). Setting them *after* the first route is registered has
> no effect (inference from the memoized getter; tests always `enable()` before the first
> request).

## 5. `app.listen`, verbs, `all`, `param`, `route`

**`app.listen`** (`lib/application.js:598-606`) is a thin wrapper over `http.createServer`:

```js
app.listen = function listen() {
  var server = http.createServer(this)           // the app IS the request handler
  var args = slice.call(arguments)
  if (typeof args[args.length - 1] === 'function') {
    var done = args[args.length - 1] = once(args[args.length - 1])
    server.once('error', done)                   // callback fires once, on listen OR error
  }
  return server.listen.apply(server, args)
}
```

It forwards all arguments to `server.listen`, so every Node signature works:
`listen(port, cb)`, `listen(port, host, backlog, cb)`, `listen()` for a random port, etc.
The `once()` wrap guarantees your callback runs exactly once — on success *or* on an
`'error'` such as `EADDRINUSE` (`test/app.listen.js:6-55`). It returns the `http.Server`, so
you can create both HTTP and HTTPS servers from one app
(`http.createServer(app)` / `https.createServer(opts, app)`, documented at
`lib/application.js:582-592`).

**Verb methods** (`get/post/put/...`) and **`app.all`** delegate to the router via
`this.route(path)` (`:471-503`); **`app.route(path)`** returns a fresh `Route` (`:256-258`);
**`app.param(name, fn)`** registers a param preprocessor and additionally supports an array
of names (`:322-334`). All of these are covered mechanistically in
[Chapter 4](04-routing-and-middleware.md).

## 6. Mounting & inheritance

`app.use(path, subApp)` mounts one app inside another. The relevant half of `app.use`
(`lib/application.js:219-241`):

```js
fns.forEach(function (fn) {
  if (!fn || !fn.handle || !fn.set) {          // :221 not an app → plain middleware
    return router.use(path, fn);
  }
  fn.mountpath = path;                          // :226
  fn.parent = this;                             // :227
  router.use(path, function mounted_app(req, res, next) {
    var orig = req.app;
    fn.handle(req, res, function (err) {        // run the sub-app
      Object.setPrototypeOf(req, orig.request)  // :233 restore parent's protos on exit
      Object.setPrototypeOf(res, orig.response) // :234
      next(err);
    });
  });
  fn.emit('mount', this);                       // :240 fire 'mount' on the sub-app
}, this);
```

A sub-app is detected by **duck typing**: `fn.handle && fn.set` (`:221`). When mounted:
- `subApp.mountpath` and `subApp.parent` are set (`:226-227`); `app.path()` walks the
  `parent`/`mountpath` chain to compute an absolute path (`:399-403`).
- The `mounted_app` wrapper runs the sub-app, then **restores the parent app's `request`/
  `response` prototypes** on `req`/`res` before returning to the parent stack (`:233-234`) —
  so a sub-app's request/response extensions don't leak upward after it finishes.
- The sub-app receives a `'mount'` event (`:240`), whose default handler
  (`lib/application.js:109-122`) links the child's prototypes to the parent's:

```js
this.on('mount', function onmount(parent) {
  // trust proxy inherit back-compat (see below)
  if (this.settings[trustProxyDefaultSymbol] === true
    && typeof parent.settings['trust proxy fn'] === 'function') {
    delete this.settings['trust proxy'];
    delete this.settings['trust proxy fn'];
  }
  Object.setPrototypeOf(this.request,  parent.request)   // :118 inherit req extensions
  Object.setPrototypeOf(this.response, parent.response)  // :119 inherit res extensions
  Object.setPrototypeOf(this.engines,  parent.engines)   // :120 inherit engines
  Object.setPrototypeOf(this.settings, parent.settings)  // :121 inherit settings
});
```

So configuration and prototype extensions flow **downward by live prototype links** — a
child reads a parent's setting unless the child has its own (which shadows it,
`test/config.js:82-101`).

### The trust-proxy inheritance dance

`defaultConfiguration` performs a deliberate two-step (`lib/application.js:99` then
`:102-105`): `set('trust proxy', false)` runs the compile switch, which sets the symbol
`@@symbol:trust_proxy_default` to **`false`** (`:374-377`); then a `defineProperty`
immediately re-asserts it to **`true`**. The net meaning: *"this app still has the default
trust-proxy config."* On mount, if the child still has the default (`symbol === true`) **and**
the parent has explicitly configured a real `trust proxy fn`, the child deletes its own
`trust proxy` keys so it inherits the parent's via the prototype chain (`:111-115`). But a
child that *explicitly* called `app.set('trust proxy', …)` — which flips its symbol to
`false` — keeps its own value (`test/config.js:103-140`). This exists purely for
backward-compatible inheritance semantics (comment: `"trust proxy inherit back-compat"`).

## 7. Rendering (overview)

`app.render(name, options?, callback)` (`lib/application.js:522-575`) resolves a template
and renders it. The essentials (full treatment in [Chapter 7](07-views-and-rendering.md)):
- **Locals precedence** (`:536`): `renderOptions = { ...this.locals, ...opts._locals, ...opts }`
  → `app.locals` < `opts._locals` < `opts` (caller-passed options win).
- **Cache**: `renderOptions.cache` defaults to the `view cache` setting (`:539-541`); when on,
  `View` objects are cached by the **name argument** (not the resolved path) (`:544-570`).
- **Lookup failure**: if the resolved `view.path` is falsy, it builds
  `Error('Failed to lookup view "<name>" in views <dir(s)>')`, attaches `err.view`, and calls
  the callback (`:558-565`) — with singular vs. plural directory phrasing depending on whether
  `views` is an array (`test/app.render.js:82-93,187-202`).
- **Sync-throw safety**: `tryRender` wraps `view.render` in try/catch so a synchronous engine
  throw reaches the callback rather than crashing (`:625-631`).

## 8. `app.engine`, locals, exports

**`app.engine(ext, fn)`** (`lib/application.js:294-308`) registers a template engine.
Throws `Error('callback function required')` if `fn` isn't a function (`:295-297`),
normalizes the extension to have a leading dot (`:300-302`), and stores it in
`this.engines['.ext']`. → [Chapter 7](07-views-and-rendering.md).

**`app.locals`** is a null-prototype object holding app-wide template variables; it always
contains `settings` (pointing at the same `this.settings`, `:131`), so templates can read
config. **`res.locals`** (per-request) is created in `app.handle` (`:173-175`). Both are
merged into rendering.

**Exports** (`lib/express.js:62-81`, verified by `test/exports.js`):

| Export | Is | Notes |
|---|---|---|
| `express` (default) | `createApplication` | The factory. |
| `express.application` | `proto` (`lib/application.js`) | Mutable; changes affect new apps (`test/exports.js:37-40`). |
| `express.request` | base `req` prototype | Mutable — extend all requests globally. |
| `express.response` | base `res` prototype | Mutable — extend all responses globally. |
| `express.Router` | `require('router')` | Standalone router factory. |
| `express.Route` | `Router.Route` | Standalone route class. |
| `express.json/raw/text/urlencoded` | `body-parser` methods | Body parsers (arity 1). → [Chapter 8](08-bundled-middleware.md). |
| `express.static` | `serve-static` | Static file middleware (arity 2). |

## 9. Traced example: creating and configuring an app

```js
const express = require('express');   // → createApplication (lib/express.js:27)
const app = express();                // createApplication(): builds fn, mixes protos,
                                       //   app.request/app.response, app.init() → defaults
app.set('view engine', 'ejs');        // settings['view engine']='ejs' (no compile switch)
app.set('trust proxy', 1);            // settings['trust proxy']=1;
                                       //   compileTrust(1) → fn(a,i){return i<1};
                                       //   settings['trust proxy fn']=that fn;
                                       //   trust_proxy_default symbol → false
app.enable('json escape');            // settings['json escape']=true
app.get('view engine');               // arguments.length===1 → returns 'ejs' (getter)
app.get('/', (req, res) => res.send('hi'));  // 2 args → route.get registered on the router
app.listen(3000);                     // http.createServer(app).listen(3000)
```

When `GET /` arrives: `app(req,res)` → `app.handle` sets `X-Powered-By: Express`, links
`req.res`/`res.req`, swaps prototypes, creates `res.locals`, and calls `router.handle`,
which matches the `/` route and runs the handler. Because `trust proxy` is now `1`, inside
the handler `req.ip` would trust exactly one proxy hop.

## 10. Errors, edge cases & gotchas

- `app.use()` with no middleware → `TypeError('app.use() requires a middleware function')`
  (`:212-214`).
- `app.engine(ext, null)` → `Error('callback function required')` (`:295-297`).
- `app.set('etag', 'bogus')` → `TypeError: unknown value for etag function: bogus` at set
  time (`lib/utils.js:148`).
- `app.get('x')` reads setting `x`; forgetting the handler when you meant a route silently
  registers nothing and returns the (probably `undefined`) setting.
- View cache key is the raw render **name**, not the resolved path — two names resolving to
  one file cache as two entries (`:545,569`).
- Default `views` dir is CWD-relative and fixed at `express()` time (`:135`).
- Concurrent view-cache misses may each build a `View` (no dedupe/locking); results converge,
  work is duplicated (inference from `:544-570`).

## Where to look

- `lib/express.js:36-56` — the factory and app shape.
- `lib/application.js:59-141` — `init` + `defaultConfiguration` (all defaults).
- `lib/application.js:152-178` — `app.handle` (per-request decoration + dispatch).
- `lib/application.js:351-380` — settings API + compilation.
- `lib/application.js:109-122,190-244` — mounting & inheritance.
- `test/config.js`, `test/app.js`, `test/app.listen.js`, `test/app.render.js` — the spec.

## Open questions

- The automatic `OPTIONS` `Allow`-list computation and 404-vs-405 behavior live in the
  external `router` package; observable behavior is pinned by `test/app.options.js` and
  covered in [Chapter 4](04-routing-and-middleware.md).

**Next:** [04 · Routing & Middleware](04-routing-and-middleware.md).
