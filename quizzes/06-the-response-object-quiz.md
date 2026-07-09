# Quiz · 06 — The Response Object

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [06 · The Response Object](../06-the-response-object.md).

**Q1.** `[Recall]` `res.status(code)` validates its argument. What happens on `res.status(99)`?
- A. Silently clamps to 100
- B. Throws a `RangeError` (must be 100–999)
- C. Throws a `TypeError`
- D. Sets status to 99

**Q2.** `[Apply]` `res.send({ user: 'tj' })` — trace what happens.
- A. Sends `[object Object]`
- B. Delegates to `res.json(...)`
- C. Throws
- D. Sends an empty body

**Q3.** `[Apply]` If `req.fresh` is true when you call `res.send(body)`, what status does the response get, and what happens to the body?
*(short answer)*

**Q4.** `[Design]` In `res.jsonp`, the callback name is sanitized with `callback.replace(/[^\[\]\w$.]/g, '')` and the body is prefixed with `/**/`. What attack does the `/**/` prefix mitigate?
*(short answer)*

**Q5.** `[Apply]` `res.cookie('sid', 'x', { signed: true })` without a configured secret does what?
- A. Signs with an empty secret
- B. Throws — `cookieParser('secret')` is required for signed cookies
- C. Silently sends unsigned
- D. Returns `false`

**Q6.** `[Apply]` `res.sendFile('relative/path.png')` with no `root` option — what happens?
*(short answer)*

---

## Answers & explanations

**Q1 — B.** `res.status` throws `TypeError` for non-integers and `RangeError` for values outside 100–999 (`lib/response.js:65`). `99` is an integer but out of range → `RangeError`.

**Q2 — B.** In `res.send`, a non-null, non-Buffer object falls through to `return this.json(chunk)` (`lib/response.js:150`). `res.json` stringifies and sets `Content-Type: application/json`.

**Q3.** `res.send` sets the status to **304** via `if (req.fresh) this.status(304)`, then (because 304) strips `Content-Type`/`Content-Length`/`Transfer-Encoding` and sends an **empty** body (`lib/response.js:185`).

**Q4.** "Rosetta Flash" JSONP abuse — the leading `/**/` comment prevents the response from being interpreted as a valid Flash/other polyglot payload when reflected as JSONP (`lib/response.js:295`). The callback sanitization and `X-Content-Type-Options: nosniff` are the companion mitigations.

**Q5 — B.** `if (signed && !secret) throw new Error('cookieParser("secret") required for signed cookies')` (`lib/response.js` `res.cookie`). Signing needs a secret from `cookie-parser`.

**Q6.** It throws a `TypeError`: `res.sendFile` requires the path to be absolute or an explicit `root` option — `if (!opts.root && !pathIsAbsolute(path)) throw new TypeError(...)` (`lib/response.js`). This prevents ambiguous relative-path resolution.
