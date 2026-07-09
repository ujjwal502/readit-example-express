# Quiz · 07 — Views & Rendering

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [07 · Views & Rendering](../07-views-and-rendering.md).

**Q1.** `[Apply]` What is the call chain when you do `res.render('email', data)`?
- A. `res.render` → engine directly
- B. `res.render` → `app.render` → `View` → engine
- C. `res.render` → `res.sendFile`
- D. `res.render` → `res.json`

**Q2.** `[Recall]` For a `.ejs` view, how does `View` obtain the engine when it isn't already registered via `app.engine`?
- A. Hardcodes EJS
- B. `require('ejs').__express`
- C. Reads a config file
- D. Uses `res.type`

**Q3.** `[Design]` `View.prototype.render` forces the callback to run via `process.nextTick` when the engine called back **synchronously**. Why normalize sync engines to async?
*(short answer)*

**Q4.** `[Apply]` `res.render('missing')` where the file can't be found — how does the error reach you?
*(short answer)*

**Q5.** `[Apply]` With `view cache` enabled, what is cached, and keyed by what?
- A. The rendered HTML, keyed by data
- B. The resolved `View` instance, keyed by view name
- C. Nothing
- D. The engine module

---

## Answers & explanations

**Q1 — B.** `res.render` merges `res.locals` and calls `app.render(view, opts, done)` (`lib/response.js` `res.render`); `app.render` constructs a `View` and calls `view.render` which invokes the engine (`lib/application.js:470`, `lib/view.js:130`).

**Q2 — B.** `View` loads `require(mod).__express` for the extension and throws if it isn't a function (`lib/view.js:78`). Engines expose `__express` as the Express-compatible entry point.

**Q3.** So the callback is **always** asynchronous, giving engines a single, predictable contract (never sometimes-sync, sometimes-async). Mixed sync/async callbacks are a classic source of subtle bugs; `process.nextTick` (`lib/view.js:145`) guarantees consistency.

**Q4.** `app.render` builds the `View`; if `view.path` is unset (lookup failed) it calls `done(err)` with a "Failed to lookup view" error (`lib/application.js:490`). Since `res.render`'s default callback does `if (err) return req.next(err)`, it propagates to your error-handling middleware.

**Q5 — B.** When `view cache` is on, `app.render` stores the resolved `View` instance in `this.cache[name]` and reuses it (`lib/application.js:480`) — it caches the *lookup/compilation*, not the rendered output.
