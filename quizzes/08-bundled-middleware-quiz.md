# Quiz · 08 — Bundled Middleware

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [08 · Bundled Middleware](../08-bundled-middleware.md).

**Q1.** `[Recall]` `express.json()` is:
- A. Implemented from scratch in `lib/`
- B. A re-export of `body-parser`'s `json`
- C. Provided by Node core
- D. An alias for `express.raw`

**Q2.** `[Recall]` `express.static(root)` maps to which package?
- A. `send`
- B. `serve-static`
- C. `finalhandler`
- D. `router`

**Q3.** `[Apply]` A client POSTs `Content-Type: text/plain` to a route behind `express.json()`. Does the JSON parser populate `req.body`?
*(short answer)*

**Q4.** `[Design]` Why does Express ship these as thin re-exports of separate packages rather than baking parsing/static-serving into core?
*(short answer)*

**Q5.** `[Recall]` Which of these is **not** bundled/re-exported by `express`?
- A. `express.urlencoded`
- B. `express.text`
- C. `express.raw`
- D. `express.cors`

---

## Answers & explanations

**Q1 — B.** `exports.json = bodyParser.json` (`lib/express.js:76`). Express delegates body parsing to `body-parser`.

**Q2 — B.** `exports.static = require('serve-static')` (`lib/express.js:79`). `serve-static` in turn builds on `send` for the actual streaming.

**Q3.** No. `body-parser`'s JSON middleware only parses when the request's `Content-Type` matches its type option (default `application/json`); for `text/plain` it skips and leaves `req.body` as its default. *(Contract is enforced inside `body-parser`; the Express side is just the re-export at `lib/express.js:76`.)*

**Q4.** It keeps core tiny and lets each parser/static server be versioned, secured, and patched independently (e.g., a `body-parser` CVE ships without an Express release). The trade-off: their internals live outside this repo. *(Inference on rationale.)*

**Q5 — D.** `express` re-exports `json`, `urlencoded`, `text`, `raw`, and `static` (`lib/express.js:76`–`:81`). There is no bundled `express.cors` — CORS is a third-party middleware.
