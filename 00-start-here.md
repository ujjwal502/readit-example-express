# 00 · Start Here

> **What you'll be able to answer after this chapter**
> - What is Express and what problem does it solve? (Purpose & domain)
> - How do I run, test, and lint it locally? (Operations)
> - What are the 5 concepts I must hold in my head to reason about the whole system? (Architecture)
> - Where does everything live, and where would my first change go?

---

## What is this?

**Express** is a "fast, unopinionated, minimalist web framework for Node.js"
(`package.json:3`, `Readme.md:3`). It is a thin, ergonomic layer on top of Node's
built-in `node:http` server. You give it a tree of **middleware** and **route handlers**
— plain `(req, res, next)` functions — and it dispatches each incoming HTTP request
through them, while decorating Node's raw `req`/`res` objects with dozens of convenience
methods (`res.json()`, `res.redirect()`, `req.accepts()`, `res.sendFile()`, …).

This repository is **Express 5.2.1** (`package.json:4`). Two facts about v5 shape the
entire codebase and are worth internalizing immediately:

1. **The core is tiny.** All of Express's own logic lives in `lib/`, which is **6 files,
   ~2,765 lines** (`lib/express.js`, `lib/application.js`, `lib/request.js`,
   `lib/response.js`, `lib/view.js`, `lib/utils.js`). Everything else is delegated to
   focused, separately-versioned npm packages.
2. **Routing is not in this repo.** In Express 5 the router was extracted into a
   standalone package, `router` (`package.json`, dependency `"router": "^2.2.0"`).
   `lib/application.js` `require`s it (`lib/application.js:26`) and delegates almost all
   routing to it. So when you read about "the routing stack" in this book, the *mechanism*
   lives in the external `router` package; this book grounds its **contract** in Express's
   own usage and in the test suite (`test/Router.js`, `test/Route.js`, `test/app.router.js`).

Express is used by essentially anyone building an HTTP server, JSON API, website, or
proxy in Node.js. Its design goal (`Readme.md:119`) is *"small, robust tooling for HTTP
servers"* that does not force an ORM, template engine, or project structure on you.

## Run it locally

Everything below is taken from `package.json` (`scripts`, `engines`) and the CI config
(`.github/workflows/ci.yml`), not guessed.

```bash
# Prerequisite: Node.js >= 18   (package.json:  "engines": { "node": ">= 18" })
# CI runs the suite on Node 18–26, ubuntu + windows (.github/workflows/ci.yml).

npm install            # install dependencies + devDependencies

npm test               # run the full test suite
#   -> mocha --require test/support/env --reporter spec --check-leaks test/ test/acceptance/

npm run lint           # eslint .   (config: .eslintrc.yml)
npm run lint:fix       # eslint . --fix

npm run test-cov       # nyc coverage, HTML + text report
npm run test-ci        # nyc coverage, lcov (what CI uses)
```

- The test runner is **Mocha** with **supertest** for HTTP-level assertions
  (`package.json` devDependencies). `--check-leaks` fails the run if a test leaks a
  global variable — a deliberate hygiene guard.
- `test/support/env.js` is auto-required before every run (it sets `NODE_ENV=test`, which
  is why the default error handler stays quiet during tests — see
  `lib/application.js:617`).

To run one of the 25+ runnable examples (each is a real, self-contained Express app):

```bash
node examples/hello-world      # then GET http://localhost:3000
node examples/content-negotiation
node examples/mvc
```

The examples double as living documentation and are themselves smoke-tested from
`test/acceptance/` (e.g. `test/acceptance/hello-world.js` boots `examples/hello-world`
and asserts on its responses).

## The mental model (the 5 things to hold in your head)

If you understand these five ideas, the rest of the codebase is detail.

### 1. `express()` returns an `app` that is a *function* — and also an EventEmitter and a settings bag

`express()` (the export of `lib/express.js`) calls `createApplication()`
(`lib/express.js:36`). The returned `app` is literally a request-handler function:

```js
// lib/express.js:37-39
var app = function(req, res, next) {
  app.handle(req, res, next);      // ← app IS the (req,res,next) listener
};
```

Onto that function it *mixes in* two prototypes with `merge-descriptors`
(`lib/express.js:41-42`): `EventEmitter.prototype` (so the app can `.emit()`/`.on()`,
used for the `'mount'` event) and the **application prototype** `proto` from
`lib/application.js` (which supplies `app.use`, `app.get`, `app.set`, `app.listen`, …).
So `app` is simultaneously: a function you can pass to `http.createServer`, an event
emitter, and an object carrying all the framework methods and a `settings` bag.

### 2. Express decorates Node's `req`/`res` by swapping their prototype at request time

Express does **not** wrap or copy Node's request/response. On every request,
`app.handle()` re-points the objects' prototype chain at Express's augmented prototypes:

```js
// lib/application.js:169-170
Object.setPrototypeOf(req, this.request)     // this.request → lib/request.js prototype
Object.setPrototypeOf(res, this.response)    // this.response → lib/response.js prototype
```

`lib/request.js` is `Object.create(http.IncomingMessage.prototype)` (`lib/request.js:30`)
and `lib/response.js` is `Object.create(http.ServerResponse.prototype)`
(`lib/response.js:43`). So `req`/`res` keep all their native behavior and *gain* Express's
methods by prototype inheritance. This is fast (no allocation per property) and is why
`req.app`, `res.locals`, etc. "just appear."

### 3. Everything is middleware, run in order by the router

An Express app is an ordered stack of functions. `app.use(fn)` and `app.get(path, fn)`
both ultimately register a function on the app's **router** (`lib/application.js:190`,
`:256`, `:471`). When a request arrives, `app.handle()` hands it to `this.router.handle()`
(`lib/application.js:177`), which walks the stack **in registration order**, calling each
matching function with `(req, res, next)`. A function advances the chain by calling
`next()`, ends the response by calling `res.send()`/`res.end()`/etc., or signals an error
with `next(err)` (which jumps to the next *error-handling* middleware — one with four
parameters `(err, req, res, next)`). Understanding "it's a stack, in order, with `next`"
explains routing, error handling, and mounting all at once. → Chapter 4.

### 4. Behavior is controlled by **settings**

`app.set(name, value)` / `app.get(name)` / `app.enable` / `app.disable`
(`lib/application.js:351`, `:420`, `:451`) manage a per-app key/value bag
(`this.settings`). Defaults are established in `defaultConfiguration()`
(`lib/application.js:90`). Settings like `'trust proxy'`, `'etag'`, `'query parser'`,
`'view engine'`, `'json spaces'`, `'x-powered-by'`, `'case sensitive routing'` change how
requests and responses behave. Some settings are *compiled* into functions on assignment
(`etag` → `etag fn`, `query parser` → `query parser fn`, `trust proxy` → `trust proxy fn`;
`lib/application.js:363-380`). → Chapter 3 and Chapter 9.

### 5. Views turn a name + data into HTML via a pluggable engine

`res.render('index', { user })` → `app.render()` → a `View` object
(`lib/view.js`) that locates a template file on disk and calls a registered **template
engine** (`app.engine(ext, fn)`, `lib/application.js:294`). Express ships no engine of its
own; it just adapts to the `(path, options, callback)` engine convention. → Chapter 7.

## Where things live

```
express/
├── index.js            # 1-line entry: module.exports = require('./lib/express')
├── lib/                # ALL of Express's own code (6 files)
│   ├── express.js      # the factory createApplication() + top-level exports
│   ├── application.js  # the `app` prototype: settings, routing delegation, listen, render, mounting
│   ├── request.js      # the `req` prototype: accepts, ip, host, protocol, query, fresh, …
│   ├── response.js     # the `res` prototype: send, json, redirect, cookie, sendFile, render, …
│   ├── view.js         # View class: template lookup + engine invocation
│   └── utils.js        # helpers: etag/query/trust compilers, content-type, methods list
├── test/               # the executable spec — 75 files, mocha + supertest
│   ├── app.*.js  req.*.js  res.*.js  Router.js  Route.js  express.*.js
│   ├── support/        # test harness (env.js, tmpl.js, utils.js)
│   ├── fixtures/       # sample views, static files, certs used by tests
│   └── acceptance/     # boots each examples/* app and asserts on it
├── examples/           # 25+ runnable example apps (auth, mvc, sessions, vhost, …)
├── History.md          # changelog — the design-rationale record (esp. the v4→v5 story)
├── Readme.md           # project overview, philosophy, team
└── .github/workflows/  # CI (ci.yml), CodeQL, OpenSSF scorecard
```

The single most useful navigation fact: **to change what a `req`/`res` method does, edit
`lib/request.js` / `lib/response.js`; to change app-level or routing behavior, edit
`lib/application.js`.** Nothing else in the repo is load-bearing at runtime except the
external dependencies.

## Your first change (a concrete example)

Say you want to add a response helper `res.sendCsv(rows)`.

1. **Where:** `lib/response.js`. It defines the `res` prototype as `res.<name> = function …`
   (e.g. `res.json` at `lib/response.js:234`). Add `res.sendCsv = function sendCsv(rows) { … }`
   in the same style, reusing `this.type('csv')` and `this.send(...)` — the existing methods
   compose.
2. **Why there:** because `app.handle()` sets every response's prototype to `this.response`,
   which derives from this file (`lib/application.js:170`, `lib/express.js:50`). Any method
   you attach to the exported `res` object is inherited by every real response.
3. **Prove it:** add `test/res.sendCsv.js` following the pattern of `test/res.json.js` —
   build a tiny app with `express()`, register a route that calls your method, and use
   `supertest` to assert the status/body/headers. Run `npm test`.

For app-level settings, follow `res.json`'s use of `app.get('json spaces')`
(`lib/response.js:239`) — read your setting via `this.app.get('my setting')` and document
its default in `defaultConfiguration()` (`lib/application.js:90`).

## Where to look

- `lib/express.js` — the factory and what Express exports.
- `lib/application.js:59` (`app.init`), `:90` (`defaultConfiguration`), `:152` (`app.handle`).
- `lib/request.js:30`, `lib/response.js:43` — the prototype roots.
- `test/` — the behavior contract for everything.

## Open questions

None at the "start here" level. Deeper mechanics (router internals, body parsing, the
send stream state machine) are external packages covered at contract level in later
chapters; where this repo cannot fully ground a behavior, that chapter says so explicitly.

**Next:** [01 · The Big Picture](01-the-big-picture.md) — the architecture, the dependency
map, and how one request flows end to end.
