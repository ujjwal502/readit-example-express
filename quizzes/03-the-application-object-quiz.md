# Quiz · 03 — The Application Object

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [03 · The Application Object](../03-the-application-object.md).

**Q1.** `[Recall]` What is the default value of the `etag` setting after `defaultConfiguration()`?
- A. `false`
- B. `'strong'`
- C. `'weak'`
- D. unset

**Q2.** `[Apply]` `view cache` is only enabled by default under one condition. Which?
- A. Always
- B. When `NODE_ENV === 'production'`
- C. When `NODE_ENV === 'development'`
- D. When a view engine is registered

**Q3.** `[Design]` `app.router` is a **lazy** getter that builds the base router on first access, reading `case sensitive routing` and `strict routing` at that moment. What practical constraint does that timing create for users?
*(short answer)*

**Q4.** `[Apply]` `app.listen(3000, cb)` — what does Express do with `cb` beyond passing it to the server?
- A. Nothing extra
- B. Wraps it with `once` and also attaches it to the server's `'error'` event
- C. Calls it twice (start + ready)
- D. Ignores it unless `NODE_ENV=production`

**Q5.** `[Design]` `cache`, `engines`, and `settings` are created with `Object.create(null)`. Why not a plain `{}`?
*(short answer)*

---

## Answers & explanations

**Q1 — C.** `this.set('etag', 'weak')` in `defaultConfiguration` (`lib/application.js:91`), which also compiles the weak-ETag function.

**Q2 — B.** `if (env === 'production') this.enable('view cache')` (`lib/application.js:135`). In development, views are re-resolved each render so edits show up without a restart.

**Q3.** Those two settings only take effect if set **before** the router is first used (before the first route/middleware registration or first request), because the router reads them once at construction (`lib/application.js:64`). Setting them after the router exists has no effect.

**Q4 — B.** `app.listen` wraps the callback with `once(...)` and does `server.once('error', done)` (`lib/application.js:560`), so a bind failure like `EADDRINUSE` reaches your callback exactly once instead of throwing uncaught.

**Q5.** `Object.create(null)` yields a prototype-less map, so a user key like `constructor` or `__proto__` can't collide with `Object.prototype` members (`lib/application.js:60`). A deliberate safety choice for user-controlled keys.
