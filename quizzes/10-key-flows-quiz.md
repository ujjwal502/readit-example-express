# Quiz · 10 — Key Flows

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [10 · Key Flows](../10-key-flows.md).

**Q1.** `[Apply]` Put the startup flow in order:
1. `http.createServer(this)`  2. `express()` builds the app  3. `app.init()` sets defaults  4. `server.listen(port)`
- A. 2 → 3 → 1 → 4
- B. 1 → 2 → 3 → 4
- C. 2 → 1 → 3 → 4
- D. 3 → 2 → 1 → 4

**Q2.** `[Apply]` For an incoming request, order the first things `app.handle` does:
1. `this.router.handle(req, res, done)`  2. set `X-Powered-By`  3. swap `req`/`res` prototypes  4. build default `done` (finalhandler)
- A. 4 → 2 → 3 → 1
- B. 1 → 2 → 3 → 4
- C. 2 → 4 → 1 → 3
- D. 3 → 1 → 2 → 4

**Q3.** `[Apply]` Trace a `GET /users/42` matched to `app.get('/users/:id', h)`: where does `req.params.id = '42'` get set?
*(short answer)*

**Q4.** `[Design]` In the JSON-POST flow, `express.json()` runs as middleware *before* your handler. Why is body parsing middleware rather than something `res.json` does automatically?
*(short answer)*

**Q5.** `[Apply]` In the error-propagation flow, a handler calls `next(err)`. With no error-handling middleware registered, who produces the response?
- A. `res.json`
- B. `finalhandler`
- C. The `router` throws
- D. Node's default

---

## Answers & explanations

**Q1 — A.** `express()` builds the app and calls `app.init()` (`lib/express.js:36`, `lib/application.js:58`); `app.listen` then does `http.createServer(this)` and `server.listen(...)` (`lib/application.js:560`).

**Q2 — A.** `app.handle` builds `done` (finalhandler) first, sets `X-Powered-By`, swaps prototypes, then calls `this.router.handle` (`lib/application.js:146`–`:170`).

**Q3.** Inside the `router` package's match step — it parses the `:id` segment and assigns `req.params.id` before invoking the matched route's handler. Express core hands off at `this.router.handle(req, res, done)` (`lib/application.js:170`); the param parsing itself is the router dependency's job.

**Q4.** Because parsing is a per-route, opt-in concern with cost and config (size limits, content types), and not every route wants a parsed body. Making it middleware lets you apply it selectively and configurably; `res.json` is about *sending*, a separate direction. *(Inference on rationale.)*

**Q5 — B.** `next(err)` with an exhausted stack invokes `done(err)`, which is `finalhandler` when no callback was passed to `app.handle` (`lib/application.js:146`) — it renders the error response and logs via `logerror` (silenced in `env === 'test'`).
