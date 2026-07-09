# 🎓 Final Exam — Express 5 Codebase

> Interleaved across all chapters. Try it fully closed-book, then check the key. If you
> can answer these, you understand how Express 5 actually works. Back to
> [all quizzes](README.md).

## Part A — Concepts & architecture

**Q1.** `[Recall]` Express 5's core is best described as:
- A. A batteries-included MVC framework
- B. A thin decoration-and-dispatch layer over `node:http` that delegates routing, parsing, and static serving
- C. A reimplementation of Node's HTTP stack
- D. A router library only

**Q2.** `[Apply]` Name the two things `express()` mixes onto the `app` function, and why each is needed. *(short answer)*

**Q3.** `[Design]` Give one concrete benefit and one concrete cost of Express 5 delegating routing/parsing/static to separate packages. *(short answer)*

## Part B — Request/response mechanics

**Q4.** `[Apply]` `res.send('<p>hi</p>')` on a fresh request (`req.fresh === true`): what status and body are sent?
- A. `200` with the HTML
- B. `304` with an empty body
- C. `204` with the HTML
- D. `200` with an empty body

**Q5.** `[Apply]` Order the reactive settings compilation: what does `app.set('trust proxy', 'loopback')` store beyond `settings['trust proxy']`? *(short answer)*

**Q6.** `[Apply]` `res.json(obj)` when `app.get('json escape')` is enabled — what does it do to `<`, `>`, `&` in the output, and why?
- A. Nothing
- B. Escapes them to `\u003c` etc. to prevent HTML sniffing
- C. Strips them
- D. URL-encodes them

## Part C — Failure & edge cases

**Q7.** `[Apply]` Which three calls throw, and with what error type each: `res.status(1000)`, `app.use()`, `res.set('Content-Type', ['a','b'])`? *(short answer)*

**Q8.** `[Apply]` A `HEAD /page` served by a handler that calls `res.send(html)` — what does the client receive?

## Part D — Scenarios (end-to-end)

**Q9.** `[Design]` **Trace it.** A `GET /users/42` arrives at a server built with `app.get('/users/:id', h)`. Walk from `app(req, res)` to the response, naming the file/function at each hop and where `req.params.id` is set.

**Q10.** `[Design]` **Change request.** Your team wants every response to carry an `X-Request-Id` header. Where do you implement it, would you use middleware or a `res` method, and what's one thing that could break if you did it wrong?

---

## Answer key

**Q1 — B.** The composition-layer framing (`lib/express.js`, `package.json` deps).

**Q2.** `EventEmitter.prototype` (so the app can `emit`/`on`, chiefly for the `'mount'` event) and the application prototype `proto` (all the `app.*` methods) — both via `mixin` in `lib/express.js:40`–`:41`.

**Q3.** Benefit: each concern is versioned/patched independently and core stays tiny/stable. Cost: the behavior lives outside this repo, so some questions can't be answered from the Express source alone (`lib/application.js:246` delegates routing; `lib/express.js:76` re-exports parsers).

**Q4 — B.** `res.send` sets `304` on freshness then strips body/headers (`lib/response.js:185`).

**Q5.** It also stores `settings['trust proxy fn'] = compileTrust('loopback')` and flips the trust-proxy back-compat symbol to `false` (`lib/application.js:305`). `'loopback'` compiles via `proxy-addr` to trust loopback addresses.

**Q6 — B.** `json escape` makes `stringify` replace `<`, `>`, `&` with `\u003c`/`\u003e`/`\u0026` so the JSON can't be sniffed as HTML when served inline (`lib/response.js:1000` `stringify`).

**Q7.** `res.status(1000)` → `RangeError` (100–999 only, `lib/response.js:65`); `app.use()` with no function → `TypeError` (`lib/application.js:196`); `res.set('Content-Type', ['a','b'])` → `TypeError('Content-Type cannot be set to an Array')` (`lib/response.js:560`).

**Q8.** Headers only, no body — `res.send` calls `this.end()` (not `this.end(chunk)`) for `HEAD` (`lib/response.js:210`).

**Q9.** `app(req,res)` → `app.handle` (`lib/application.js:146`): builds `finalhandler` `done`, sets `X-Powered-By`, swaps `req`/`res` prototypes, then `this.router.handle(req, res, done)` (`:170`). The **`router` package** matches `GET /users/:id`, sets `req.params.id = '42'`, and calls `h(req, res, next)`. `h` calls e.g. `res.json(user)` → `res.send` → `res.end` sends bytes (`lib/response.js:120`). If `h` had called `next()` and the stack emptied, `done()` → `finalhandler` 404.

**Q10.** Implement it as **middleware** registered early with `app.use((req, res, next) => { res.set('X-Request-Id', id); next() })`, because it must run before handlers and apply to all routes. (A `res` helper would still require every handler to call it.) A thing that could break: **setting headers after the response has started** — if you add it after a handler already called `res.send`/`res.end`, `setHeader` throws "headers already sent." Middleware ordering (before the route) avoids that.
