# Quiz · 05 — The Request Object

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [05 · The Request Object](../05-the-request-object.md).

**Q1.** `[Recall]` `req.get('Referrer')` is special-cased. What does it return?
- A. Only the `Referrer` header
- B. Either the `referrer` or `referer` header (interchangeable)
- C. Always `undefined`
- D. The `Origin` header

**Q2.** `[Apply]` `req.query` is a getter. What decides *how* the raw query string is parsed?
- A. It always uses `JSON.parse`
- B. The compiled `query parser fn` from the `query parser` setting
- C. Node's `url` module only
- D. The `Accept` header

**Q3.** `[Apply]` Behind a trusted proxy, how does `req.protocol` decide between `http` and `https`?
*(short answer)*

**Q4.** `[Design]` `req.ip` and `req.host` both consult `trust proxy fn` before believing `X-Forwarded-*` headers. Why gate on trust rather than always reading those headers?
*(short answer)*

**Q5.** `[Apply]` For `req.fresh` to possibly be `true`, which must hold?
- A. Any method, any status
- B. Method is GET/HEAD **and** status is 2xx or 304
- C. Only POST requests
- D. Status must be 500

---

## Answers & explanations

**Q1 — B.** The `referer`/`referrer` case returns `this.headers.referrer || this.headers.referer` (`lib/request.js:70`) — the two spellings are treated as interchangeable.

**Q2 — B.** The `query` getter calls `this.app.get('query parser fn')`; if disabled it returns an empty null-proto object, otherwise it parses `parse(this).query` with that function (`lib/request.js:215`). The function comes from `compileQueryParser` (`simple`→`querystring.parse`, `extended`→`qs`).

**Q3.** It starts from `this.socket.encrypted ? 'https' : 'http'`; if `trust proxy fn` trusts the peer, it instead trusts `X-Forwarded-Proto` (taking the first comma-separated value) (`lib/request.js:275`).

**Q4.** `X-Forwarded-*` headers are client-spoofable. Trusting them unconditionally would let any client claim an arbitrary IP/host/protocol. Gating on `trust proxy fn` means they're only believed when the immediate peer is a proxy you configured as trusted (`lib/request.js` `ip`/`host` getters).

**Q5 — B.** `req.fresh` returns `false` unless the method is GET/HEAD and the status is 2xx or 304, then defers to the `fresh` module against ETag/Last-Modified (`lib/request.js:420`).
