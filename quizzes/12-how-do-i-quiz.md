# Quiz · 12 — How Do I…?

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [12 · How Do I…?](../12-how-do-i.md).

**Q1.** `[Apply]` You want a helper method on **every response** (e.g. `res.ok()`). Where do you add it?
- A. `lib/application.js`
- B. `lib/response.js`, next to the other `res.*` methods
- C. `lib/request.js`
- D. `index.js`

**Q2.** `[Apply]` You need to register a template engine for `.hbs` files. Which API, and what signature must the engine have?
*(short answer)*

**Q3.** `[Apply]` To make routing case-sensitive, which setting do you set — and what timing gotcha applies?
*(short answer)*

**Q4.** `[Recall]` How do you run the test suite for this repo?
- A. `npm start`
- B. `npm test` (mocha)
- C. `node index.js`
- D. `make`

**Q5.** `[Apply]` You want the same handlers to run for **every** HTTP method on a path. Which method do you use?
- A. `app.get`
- B. `app.use`
- C. `app.all`
- D. `app.route`

---

## Answers & explanations

**Q1 — B.** Response helpers live on the response prototype in `lib/response.js`; every live `res` gets it via the prototype swap in `app.handle` (`lib/application.js:164`).

**Q2.** `app.engine('hbs', fn)` where `fn` has the signature `(path, options, callback)` (`lib/application.js` `app.engine`). Engines that expose `.__express` are picked up automatically; otherwise map them explicitly with `app.engine`.

**Q3.** `app.set('case sensitive routing', true)` (or `app.enable(...)`). Gotcha: the base router reads it **once at first use** (`lib/application.js:64`), so set it before registering routes or handling requests, or it won't take effect.

**Q4 — B.** `npm test` runs the mocha suite (`package.json` scripts). `npm run lint`, `npm run test-cov` are the other scripted steps.

**Q5 — C.** `app.all(path, ...)` loops over `methods` and registers the handlers for every HTTP method (`lib/application.js:512`). `app.use` runs regardless of method but isn't route-scoped the same way.
