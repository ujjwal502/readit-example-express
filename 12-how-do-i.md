# 12 · How Do I…? (Cookbook)

> **What you'll be able to answer after this chapter**
> - How do I run, test, lint, and debug Express itself? (Operations)
> - Where do I make a given change, and what would break if I got it wrong? (Operations / Change)
> - How are the real-world patterns (auth, sessions, MVC, static, vhost, negotiation) wired? (Recipes)

Every recipe points at a real file. The `examples/` directory is the canonical recipe book —
each example is a runnable app, and each is smoke-tested from `test/acceptance/`.

---

## Operations: running the project

```bash
npm install                 # deps + devDeps (Node >= 18)
npm test                    # mocha spec reporter, --check-leaks, over test/ and test/acceptance/
npm run lint                # eslint .   (.eslintrc.yml)
npm run lint:fix            # eslint . --fix
npm run test-cov            # nyc coverage (HTML + text)
npm run test-ci             # nyc lcov (what CI runs)
node examples/hello-world   # run any example app
DEBUG=express:* node app.js # turn on Express's debug logging (express:application, express:view)
```

- Tests use **Mocha + supertest**; `test/support/env.js` sets `NODE_ENV=test` and
  `NO_DEPRECATION=body-parser,express` before every run (so error logs and deprecation warnings
  stay quiet). `--check-leaks` fails on leaked globals.
- CI runs on Node **18–26**, ubuntu + windows (`.github/workflows/ci.yml`).

## Where a given change goes (the change map)

| I want to change… | Edit | Because |
|---|---|---|
| A `req` method/property (e.g. `req.ip`) | `lib/request.js` | It's the `req` prototype; installed per request (`lib/application.js:169`). |
| A `res` method (e.g. `res.send`) | `lib/response.js` | It's the `res` prototype (`lib/application.js:170`). |
| App-level behavior, a new setting, mounting | `lib/application.js` | The app prototype. |
| A new default setting value | `defaultConfiguration` (`lib/application.js:90`) | Where every default is established. |
| The factory / what `express` exports | `lib/express.js` | The `createApplication` factory + exports. |
| View lookup / engine loading | `lib/view.js` | The `View` class. |
| An ETag/query/trust/content-type helper | `lib/utils.js` | The shared helpers. |
| Routing/path-matching behavior | the external `router` package | Not in this repo; open an issue/PR upstream. |
| Body parsing / static serving | `body-parser` / `serve-static` | Re-exported, not in this repo. |

**What breaks if you get it wrong:** adding a method to `lib/response.js` affects *every*
response (it's a shared prototype); changing a default in `defaultConfiguration` affects every
new app; touching `app.handle`'s prototype-swap or `res.locals` setup affects the entire request
pipeline. Always add a matching `test/*.js` and run `npm test`.

## Recipe: add a `res` helper

Follow the shape of `res.json` (`lib/response.js:234-248`). Add to `lib/response.js`:
```js
res.sendCsv = function sendCsv(rows) {
  this.type('csv');                                   // reuse res.type → text/csv
  const body = rows.map(r => r.join(',')).join('\n');
  return this.send(body);                             // reuse res.send (CL/ETag/HEAD handled)
};
```
Prove it with a `test/res.sendCsv.js` modeled on `test/res.json.js` (build an app with
`express()`, register a route, assert with supertest). Because `app.handle` sets every
response's prototype to `this.response` (derived from this file), the method is inherited by all
responses.

## Recipe: add an app setting

Read it via `this.app.get('my setting')` in `req`/`res` code, and set its default in
`defaultConfiguration` (`lib/application.js:90`). Follow `json spaces`
(`lib/response.js:239`). If the setting needs to be *compiled* (like `etag`/`trust proxy`), add a
`case` to the `app.set` switch (`lib/application.js:363-380`) and a compiler in `lib/utils.js`
(follow `compileETag`).

## Recipe: routing patterns (Express 5 syntax)

```js
app.get('/', handler);                       // basic
app.get('/users/:id', handler);              // named param (single segment, decoded)
app.get('/user/:id{/:op}', handler);         // OPTIONAL segment — v5 uses {…} braces, NOT :op?
app.get('/files/*file', handler);            // splat → req.params.file is an ARRAY (join('/'))
app.get(/^\/[a-z]oo/, handler);              // regexp route (pathname only)
app.route('/users').get(list).post(create);  // multiple methods, one path
app.all('/*splat', catchAll);                // every method
app.param('id', (req,res,next,id) => {...}); // param preprocessor (fires once per value)
```
Real files: `examples/params/index.js` (params + validation), `examples/downloads/index.js:26-27`
(splat → `req.params.file.join('/')`), `examples/resource/index.js:15` (`/:a..:b{.:format}`),
`examples/route-map/index.js:14-29` (a recursive route-map DSL). See
[Chapter 4](04-routing-and-middleware.md) for full matching semantics.

## Recipe: modular routers (split routes across files)

```js
// controllers/api_v1.js
const router = require('express').Router();
router.get('/things', list);
module.exports = router;

// index.js
app.use('/api/v1', require('./controllers/api_v1'));   // mount at prefix
```
Grounded: `examples/multi-router/index.js:7-8`, `controllers/api_v1.js:5-15`. For a per-resource
split, see `examples/route-separation/index.js:36-49`.

## Recipe: middleware chains & per-route guards

```js
function loadUser(req, res, next) { /* … */ req.user = user; next(); }
function restrictToSelf(req, res, next) { req.user.id === req.session.uid ? next() : next(403); }
app.get('/user/:id', loadUser, restrictToSelf, (req, res) => res.json(req.user));
```
Middleware factories return closures: `andRestrictTo(role)` (`examples/route-middleware/index.js:50-58`).
Use `next('route')` to skip to the next matching route
(`examples/mvc/controllers/user/index.js:19`).

## Recipe: parse request bodies

```js
app.use(express.json());                          // application/json → req.body: object
app.use(express.urlencoded({ extended: true }));  // forms → nested object (extended=qs)
app.use(express.text());                          // text/plain → req.body: string
app.use(express.raw());                            // octet-stream → req.body: Buffer
```
Register **before** the routes that read `req.body`. See [Chapter 8](08-bundled-middleware.md)
for every option. Remember: default `urlencoded` is `extended:false` (flat keys only).

## Recipe: serve static files

```js
app.use(express.static(path.join(__dirname, 'public')));         // at root
app.use('/static', express.static(path.join(__dirname, 'public'))); // under a prefix
```
Grounded: `examples/static-files/index.js:22-36`. To serve one file dynamically without exposing
a directory, use `res.sendFile` (`examples/search/index.js:69-71`) or `res.download`
(`examples/downloads/index.js:26-34`). Set `fallthrough:false` to surface 400/403/405 instead of
silent 404s.

## Recipe: templating

```js
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));
app.get('/', (req, res) => res.render('index', { title: 'Home' }));
```
Custom extension: `app.engine('.html', require('ejs').__express)` then
`app.set('view engine','html')` (`examples/ejs/index.js:23-36`). Custom engine function:
`examples/markdown/index.js:17-25`. Per-controller engines (EJS + Handlebars in one app):
`examples/mvc`. Full contract in [Chapter 7](07-views-and-rendering.md).

## Recipe: content negotiation

```js
res.format({
  html: () => res.send('<b>hi</b>'),
  json: () => res.json({ msg: 'hi' }),
  default: () => res.status(406).send('Not Acceptable'),
});
```
Grounded: `examples/content-negotiation/index.js:10-27`, reusable via a handler module
(`users.js:5-19`). `res.format` sets `Vary: Accept` and the Content-Type for you.

## Recipe: error handling

```js
// throw or next(err) in any handler:
app.get('/x', (req, res) => { throw createError(404, 'not found'); });

// 404 fallback (last non-error middleware):
app.use((req, res) => res.status(404).render('404'));

// error handler (arity 4, registered LAST):
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({ error: err.message });
});
```
Grounded: `examples/error/index.js:20-47`, `examples/error-pages/index.js:63-97`,
`examples/web-service/index.js:98-111`. Use `http-errors`' `createError(status, msg)` and the
`err.status` convention. Async errors: call `next(err)` (a **resolved** promise does not
auto-advance — see [Chapter 11 §6](11-failure-modes-and-edge-cases.md#6-concurrency-ordering--idempotency)).

## Recipe: sessions & auth

```js
app.use(session({ resave: false, saveUninitialized: false, secret: '…' }));  // express-session
app.get('/count', (req, res) => { req.session.views = (req.session.views||0)+1; res.send(String(req.session.views)); });
```
Grounded: `examples/session/index.js:16-30`, `examples/cookie-sessions/index.js` (cookie-session).
Full auth flow — password hashing (`pbkdf2-password`), session regeneration to prevent fixation
(`req.session.regenerate`), a `restrict` guard, flash messages — is in
`examples/auth/index.js:8,50-119`.

## Recipe: cookies

```js
app.use(cookieParser('secret'));                 // → req.cookies / req.signedCookies
res.cookie('remember', '1', { maxAge: 900000, httpOnly: true });
res.cookie('token', 'abc', { signed: true });    // needs cookieParser secret
res.clearCookie('remember');
```
Grounded: `examples/cookies/index.js:19,35,43`. See [Chapter 6 §6](06-the-response-object.md#6-cookies-rescookie-and-resclearcookie).

## Recipe: run behind a proxy (get real client IP/proto)

```js
app.set('trust proxy', 1);          // trust one hop; or true / '10.0.0.0/8' / a function
// now req.ip, req.protocol, req.hostname reflect X-Forwarded-* from the trusted proxy
```
See [Chapter 5 §7](05-the-request-object.md#7-trust-proxydependent-properties). Default is
`false` (ignore forwarded headers) — only enable behind a proxy you control.

## Recipe: virtual hosts (multiple apps by hostname)

```js
app.use(vhost('*.example.com', subdomainApp));   // subdomainApp is a full express() instance
app.use(vhost('example.com', mainApp));
```
Grounded: `examples/vhost/index.js:37-47`. `req.vhost[0]` holds the captured subdomain.

## Recipe: extend all requests/responses globally

```js
express.response.message = function (msg) { /* … */ return this; };   // affects every app
```
`express.request`/`express.response` are the base prototypes (`lib/express.js:63-64`); mutating
them affects all apps. Grounded: `examples/mvc/index.js:24-31` (`app.response.message` per-app;
relies on session middleware being mounted first).

## Recipe: create an HTTPS server

```js
const https = require('node:https');
https.createServer({ key, cert }, app).listen(443);   // pass the app as the request handler
```
The app is just an `(req,res)` function (`lib/express.js:37-39`), so any server that takes a
listener works. `app.listen()` is only sugar for `http.createServer(app).listen()`
(`lib/application.js:598-606`).

## Recipe: write a test (the project's own pattern)

```js
const express = require('..');
const request = require('supertest');

describe('res.myThing', () => {
  it('does the thing', (done) => {
    const app = express();
    app.get('/', (req, res) => res.myThing());
    request(app).get('/').expect(200, 'expected body', done);
  });
});
```
This is the shape of every `test/res.*.js` / `test/req.*.js`. Tests are the executable spec —
mine the existing ones for the exact behavior you're changing. Acceptance tests boot an
example app and assert on it (`test/acceptance/hello-world.js`).

## The examples catalog (what each demonstrates)

| Example | Demonstrates |
|---|---|
| `hello-world` | Minimal app; `module.exports = app` + `if (!module.parent) app.listen()` idiom. |
| `auth` | Sessions, password hashing, session regeneration, `restrict` guard, flash messages. |
| `content-negotiation` | `res.format`, reusable handler modules. |
| `cookies` / `cookie-sessions` | `cookie-parser`, `cookie-session`, signed cookies. |
| `session` | `express-session` counter. |
| `downloads` | `res.download`, splat routes, 404 handling. |
| `ejs` / `markdown` / `view-constructor` | Engine wiring: `.__express`, custom engine fn, custom `View` class. |
| `mvc` | Dynamic controller loading (`boot.js`), per-controller engines, `res.message`, method-override. |
| `error` / `error-pages` | Error middleware, 404 pages, `verbose errors` toggle by env. |
| `multi-router` / `route-separation` / `route-map` | Modular routers and route DSLs. |
| `params` | `app.param`, validation with `createError`. |
| `resource` | REST resource DSL (`app.resource`). |
| `route-middleware` | Per-route middleware factories, `next('route')`. |
| `static-files` | `express.static` at root / prefix / stacked. |
| `vhost` | Virtual hosts with sub-apps. |
| `web-service` | API-key middleware mounted at `/api`, JSON error responses. |
| `search` / `online` | `res.sendFile`, real-time patterns. |

## Where to look

- `package.json` `scripts`, `.github/workflows/ci.yml` — the operations surface.
- `examples/` + `examples/README.md` — the full recipe catalog.
- `test/` — the test-writing pattern and the behavior contract.

## Open questions

- The `express-generator` (`express(1)` scaffolding CLI mentioned in `Readme.md`) is a separate
  repo, not in this checkout.

**Next:** [13 · Glossary](13-glossary.md).
