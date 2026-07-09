# Quiz · 09 — Cross-Cutting Concerns

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [09 · Cross-Cutting Concerns](../09-cross-cutting-concerns.md).

**Q1.** `[Recall]` `compileETag` maps setting values to functions. What does `'weak'` (the default) compile to?
- A. The strong ETag generator
- B. The weak ETag generator (`wetag`)
- C. `undefined`
- D. It throws

**Q2.** `[Apply]` `app.set('query parser', 'extended')` — which parser backs it?
- A. Node's `querystring.parse`
- B. `qs.parse` (with `allowPrototypes: true`)
- C. `JSON.parse`
- D. A no-op

**Q3.** `[Apply]` `app.set('trust proxy', 2)` (a number) — what trust function does `compileTrust` produce?
*(short answer)*

**Q4.** `[Design]` `compileETag`/`compileQueryParser`/`compileTrust` run at `app.set` time, not per request. Why compile eagerly?
*(short answer)*

**Q5.** `[Recall]` An unknown value like `app.set('etag', 'banana')` does what?
- A. Falls back to weak
- B. Throws a `TypeError: unknown value for etag function`
- C. Disables ETags
- D. Uses strong

---

## Answers & explanations

**Q1 — B.** `compileETag` maps `true`/`'weak'` → `exports.wetag`, `'strong'` → `exports.etag`, `false` → none (`lib/utils.js:130`).

**Q2 — B.** `compileQueryParser` maps `'extended'` → `parseExtendedQueryString`, which calls `qs.parse(str, { allowPrototypes: true })` (`lib/utils.js:160`, `:265`). `'simple'` uses `querystring.parse`.

**Q3.** A hop-count function: `function(a, i){ return i < val }` — trust the first `val` proxy hops (`lib/utils.js:190`). Numbers mean "trust N hops"; booleans, strings (CSV), and arrays compile differently via `proxy-addr`.

**Q4.** Because these run on the hot path (every request reads `etag fn`, `query parser fn`, `trust proxy fn`). Compiling once at configuration time avoids reparsing the setting on every request. The compiled fn is stored under a `<key> fn` setting (`lib/application.js:295`).

**Q5 — B.** `compileETag`'s `switch` throws `TypeError('unknown value for etag function: ' + val)` for unrecognized values (`lib/utils.js:150`) — fail fast on misconfiguration rather than silently doing the wrong thing.
