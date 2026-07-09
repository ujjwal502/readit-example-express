# Quiz · 11 — Failure Modes & Edge Cases

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [11 · Failure Modes & Edge Cases](../11-failure-modes-and-edge-cases.md).

**Q1.** `[Apply]` Match the call to its thrown error:
1. `res.status('200')`  2. `res.status(1000)`  3. `res.set('Content-Type', ['a','b'])`
- A. `RangeError`   B. `TypeError` (Content-Type array)   C. `TypeError` (not an integer)

**Q2.** `[Apply]` `req.get()` with no argument does what?
- A. Returns all headers
- B. Throws `TypeError` — name is required
- C. Returns `undefined`
- D. Returns `''`

**Q3.** `[Apply]` A `HEAD` request reaches `res.send(body)`. What goes over the wire?
*(short answer)*

**Q4.** `[Design]` `res.status` throws on a bad code instead of clamping. In an on-call scenario, why is throwing arguably safer than silently coercing?
*(short answer)*

**Q5.** `[Apply]` `res.send` on a `204` or `304` response — what does it do to `Content-Type` / `Content-Length`?
- A. Leaves them as-is
- B. Strips `Content-Type`, `Content-Length`, and `Transfer-Encoding`, sends empty body
- C. Sets `Content-Length: 0` only
- D. Throws

---

## Answers & explanations

**Q1.** 1→C (non-integer → `TypeError`), 2→A (out of 100–999 → `RangeError`), 3→B (`res.set` throws `TypeError('Content-Type cannot be set to an Array')`). See `lib/response.js:65` and `res.set` (`lib/response.js:560`).

**Q2 — B.** `req.get` throws `TypeError('name argument is required to req.get')` when called with no name, and another `TypeError` if the name isn't a string (`lib/request.js:60`).

**Q3.** Only headers — no body. `res.send` special-cases `if (req.method === 'HEAD') this.end()` (skipping the body) instead of `this.end(chunk)` (`lib/response.js:210`), matching HTTP semantics for HEAD.

**Q4.** A bad status code is a programming error; clamping would ship a subtly-wrong response to production and hide the bug. Throwing (`lib/response.js:65`) surfaces it in tests/CI at the call site, before it reaches users. *(Inference on rationale.)*

**Q5 — B.** For `204`/`304`, `res.send` removes `Content-Type`, `Content-Length`, and `Transfer-Encoding` and sets the body to empty (`lib/response.js:185`), because those statuses must not carry a body.
