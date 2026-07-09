# Quiz · 00 — Start Here

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [00 · Start Here](../00-start-here.md).

**Q1.** `[Recall]` What does `express()` return?
- A. A plain object with route methods
- B. A JavaScript **function** `(req, res, next)` with the app methods mixed onto it
- C. An instance of a `Server` class
- D. A Node `http.Server`

**Q2.** `[Apply]` Why can you pass `app` straight to `http.createServer(app)`?
*(short answer)*

**Q3.** `[Recall]` Which of these does Express core **not** implement itself?
- A. The `res.json()` helper
- B. The settings bag (`app.set`/`app.get`)
- C. Route path matching (`:id`, wildcards)
- D. The `x-powered-by` header

**Q4.** `[Design]` The mental model says "`req`/`res` are Node's own objects, augmented." Why does Express swap the prototype of the live objects instead of wrapping them in new objects?
*(short answer)*

**Q5.** `[Apply]` You want to add a `res.sendOk()` helper available on every response. Where do you add it, and why does it "just work" for every request?
*(short answer)*

---

## Answers & explanations

**Q1 — B.** `createApplication()` builds `var app = function(req, res, next){ app.handle(...) }` and mixes the application prototype onto it (`lib/express.js:36`). *(C/D are wrong — `app.listen` *creates* an `http.Server`, but the app itself is a function; A misses that it's callable.)*

**Q2.** Because the app **is** a request-listener function with signature `(req, res)`, which is exactly what `http.createServer` expects (`lib/express.js:37`, `lib/application.js:560`). No adapter is needed.

**Q3 — C.** Path matching lives in the `router` package; core forwards to it (`lib/application.js:246`). A, B, and D are all implemented in `lib/` (`response.js`, `application.js:295`, `application.js:150`).

**Q4.** Swapping the prototype (`Object.setPrototypeOf(req, this.request)`, `lib/application.js:163`) gives every request Express's helpers *and* Node's native methods on the same object, with zero wrapper allocation per request — and nothing downstream has to know it was augmented.

**Q5.** Add it to the response prototype in `lib/response.js` (next to `res.send`, `lib/response.js:120`). Every live `res` gets `lib/response.js`'s exports set as its prototype in `app.handle` (`lib/application.js:164`), so the method is available on `res` for every request with no extra wiring.
