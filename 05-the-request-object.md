# 05 · The Request Object

> **What you'll be able to answer after this chapter**
> - Every `req` property/method: exact contract, inputs, outputs, and edge cases. (Interfaces)
> - How `trust proxy` changes `req.ip`/`ips`/`protocol`/`host`/`hostname`/`subdomains`. (Security)
> - How `req.query` behaves under each `query parser` setting, and its prototype-pollution posture. (State / Security)
> - Which `req` members are Express's vs. provided by the external router/cookie-parser. (Architecture)

**Source of truth:** `lib/request.js` (528 lines), grounded by every `test/req.*.js`.
`req` is `Object.create(http.IncomingMessage.prototype)` (`lib/request.js:30`) — Express's
methods sit on a prototype layered over Node's, installed per request by
`Object.setPrototypeOf(req, this.request)` (`lib/application.js:169`). Getters are defined via
a small helper `defineGetter(obj, name, getter)` → `Object.defineProperty(..., { configurable:
true, enumerable: true, get })` (`lib/request.js:521-527`).

Everything here is **synchronous, pure over per-request state** (headers, socket) plus
immutable app settings — no shared mutable state, no locks. Getters **recompute on every
access** (no memoization), which matters in hot handlers (see §8).

---

## 1. What's Express's vs. what's external

| Member | Defined in | Notes |
|---|---|---|
| `get`/`header`, `accepts*`, `is`, `range`, `query`, `protocol`, `secure`, `ip`, `ips`, `host`, `hostname`, `subdomains`, `path`, `fresh`, `stale`, `xhr` | `lib/request.js` | This chapter. |
| `params`, `route`, `baseUrl`, `originalUrl`, `url` mutation | external `router` package | See [Chapter 4](04-routing-and-middleware.md). |
| `cookies`, `signedCookies`, `secret` | external `cookie-parser` | Only present when you `app.use(cookieParser(secret))`. |
| `body` | external `body-parser` | Only after a body parser runs. See [Chapter 8](08-bundled-middleware.md). |
| `app`, `res`, `next` | wired by Express | `app` (`lib/express.js:46`), `res` (`lib/application.js:165`), `next` (by router during dispatch). |

## 2. `req.get(name)` / `req.header(name)` — read a header

```js
// lib/request.js:63-83
req.get = req.header = function header(name) {
  if (!name) throw new TypeError('name argument is required to req.get');       // :65-67
  if (typeof name !== 'string') throw new TypeError('name must be a string to req.get'); // :69-71
  var lc = name.toLowerCase();
  switch (lc) {
    case 'referer':
    case 'referrer':
      return this.headers.referrer || this.headers.referer;   // interchangeable
    default:
      return this.headers[lc];
  }
};
```

- Case-insensitive; returns `undefined` for absent headers (`test/req.get.js:13`).
- **`Referer`/`Referrer` are interchangeable** — either spelling reads either header
  (`test/req.get.js:23-34`).
- **Throws** on falsy name or non-string name (`test/req.get.js:36-58`) → surfaces as HTTP
  **500** (the router converts the throw to `next(err)` → finalhandler).

## 3. Content negotiation: `accepts`, `acceptsEncodings`, `acceptsCharsets`, `acceptsLanguages`

All four build a fresh `accepts(this)` negotiator (from the `accepts` package) and delegate
(`lib/request.js:127-187`). **Shared contract:** with a list/array they return the best match
or `false`; when the relevant `Accept*` header is **absent**, everything is acceptable so the
first candidate is returned.

- **`req.accepts(...types)`** (`:127-130`): single string, arg-list, or array. Absent `Accept`
  → first candidate truthy (`test/req.accepts.js:8-18`); a match returns the matched token;
  no match → `false` (`:33-44`); quality-weighted best match honored — `*/html;q=.5,
  application/json` → `application/json` (`:99-110`); canonicalizes extension names to full
  types (`:112-123`).
- **`req.acceptsEncodings(...enc)`** (`:140-143`): `gzip, deflate` → `'gzip'`; unknown →
  `false` (`test/req.acceptsEncodings.js:8-37`).
- **`req.acceptsCharsets(...charsets)`** (`:171-174`): best match driven by the client header's
  order — args `('utf-8','iso-8859-1')` with `Accept-Charset: iso-8859-1, utf-8` →
  `'iso-8859-1'` (`test/req.acceptsCharsets.js:49-60`); absent header → truthy first.
- **`req.acceptsLanguages(...langs)`** (`:185-187`): `en;q=.5, en-us` → `acceptsLanguages('en-us')
  === 'en-us'`, `('es') === false`; absent → every arg echoed (`test/req.acceptsLanguages.js`).

These never throw for bad input — they return `false`. Pair them with a 406 response when the
result is falsy (that's what `res.format` does, [Chapter 6](06-the-response-object.md)).

## 4. `req.is(...types)` — content-type matching

```js
// lib/request.js:269-281  (delegates to type-is)
req.is = function is(types) {
  var arr = types;
  if (!Array.isArray(types)) { arr = new Array(arguments.length); for (...) arr[i]=arguments[i]; }
  return typeis(this, arr);
};
```

Returns the matched type string, `false` on mismatch, or `null` when the request has no body
/ content-type (the `null` path is documented at `:265` but not exercised by tests —
inference). Grounded (`test/req.is.js`): exact `application/json` → `"application/json"`
(`:8-20`); mismatch → `false`; **no Content-Type → `false`** (`:51-64`); extension `'json'` →
`"json"`; wildcards `*/json`, `application/*` match (`:82-140`); charset in the header is
ignored.

## 5. `req.range(size, options)` — Range header parsing

```js
// lib/request.js:214-218  (delegates to range-parser)
req.range = function range(size, options) {
  var range = this.get('Range');
  if (!range) return;                          // absent header → undefined
  return parseRange(size, range, options);
};
```

- Absent `Range` → `undefined` (`test/req.range.js:73-83`).
- Returns an array with a `.type` property (the unit, e.g. `'bytes'` or an arbitrary
  `'users'`); each element `{start, end}` is **inclusive** and `end` is capped to `size-1`
  (`bytes=0-100` with size 75 → `[{0,74}]`, `:21-32`).
- `{ combine: true }` merges overlapping/adjacent ranges (`bytes=0-50,51-100` → `[{0,100}]`,
  `:86-103`).
- Per the docstring (`:191-196`): `-1` unsatisfiable, `-2` syntactically invalid (from
  range-parser; not covered by these tests).

## 6. `req.query` and the query-parser setting

```js
// lib/request.js:230-241
defineGetter(req, 'query', function query(){
  var queryparse = this.app.get('query parser fn');
  if (!queryparse) return Object.create(null);   // parsing disabled → null-proto empty object
  var querystring = parse(this).query;           // parseurl (cached)
  return queryparse(querystring);
});
```

The `query parser fn` is compiled from the `query parser` setting
(`lib/utils.js:162-184`, [Chapter 9](09-cross-cutting-concerns.md)):

| Setting | `query parser fn` | Behavior | Grounded |
|---|---|---|---|
| `'simple'` / `true` (default) | `node:querystring.parse` | Flat keys only; `?user[name]=tj` → `{ "user[name]": "tj" }` | `test/req.query.js:17-23` |
| `'extended'` | `qs.parse(str, { allowPrototypes: true })` | Nested objects/arrays; `foo[0][bar]=…` → nested | `:25-32` |
| `false` | `undefined` | Parsing **disabled** → `Object.create(null)` | `:65-73` |
| a `function` | that function | Receives the raw query string | `:53-63` |
| unknown string | — | **Throws** `TypeError: unknown value for query parser function: …` at `app.set` time | `:85-90` |

- **Default is `'simple'`** (`lib/application.js:97`) — an Express 5 change from v4's
  `'extended'`, chosen to reduce surprise and shrink the `qs` attack surface.
- **Prototype-pollution posture:** the disabled path returns a null-prototype object (safe),
  but `'extended'` deliberately sets `allowPrototypes: true` (`lib/utils.js:269`), which
  permits `__proto__`-style keys into the parsed object — a documented `qs` trade-off and a
  footgun if you enable extended parsing on untrusted input.
- **Key-count bound:** the `'simple'` parser is Node's `querystring.parse`, which silently caps
  at **1000 keys by default** (`maxKeys`), dropping extras rather than erroring — so a query with
  thousands of parameters won't blow up memory, but keys beyond the cap silently vanish. The
  `'extended'` parser (`qs`) has its own `parameterLimit`/`depth` bounds. (Node/`qs` behavior;
  Express passes the raw string straight through, `lib/request.js:238-240`.)

## 7. Trust-proxy–dependent properties

Six getters change behavior based on the compiled `trust proxy fn`. That function is built by
`compileTrust` (`lib/utils.js:194-214`):

| `trust proxy` value | Meaning |
|---|---|
| `false` / unset (default) | Trust nothing → ignore all `X-Forwarded-*`. |
| `true` | Trust all proxies. |
| a number `N` | Hop count: trust the `N` closest proxies (`fn(addr,i) => i < N`). |
| comma-string / array | Trust these IPs/subnets (`proxyaddr.compile`). |
| a `function` | Used as the predicate directly. |

Default is `false` (`lib/application.js:99`) — so **out of the box, forwarded headers are not
trusted**, which is the safe default. Sub-apps inherit the parent's trust fn unless they set
their own (`lib/application.js:109-115`; [Chapter 3 §6](03-the-application-object.md#the-trust-proxy-inheritance-dance)).

### `req.protocol` (`:297-315`) and `req.secure` (`:326-328`)
Base protocol is `socket.encrypted ? 'https' : 'http'`. If the socket's peer is **not**
trusted, that base value is returned and `X-Forwarded-Proto` is **ignored**
(`test/req.protocol.js:98-111`, `:51-64`). If trusted, `X-Forwarded-Proto` is used; a
comma-separated value takes the **leftmost** (original client-facing) hop, trimmed
(`:310-314`). `req.secure` is simply `req.protocol === 'https'`. Proof of the comma rule via
`secure`: `X-Forwarded-Proto: 'http, https'` → `secure` false; `'https, http'` → `secure`
true (`test/req.secure.js:53-81`).

### `req.ip` (`:340-343`) and `req.ips` (`:357-366`)
- `req.ip = proxyaddr(this, trust)` — the socket address unless trust promotes an
  `X-Forwarded-For` entry. With trust-all and `X-Forwarded-For: client, p1, p2` → `'client'`
  (`test/req.ip.js:10-23`); hop-count `2` → `'p1'` (`:25-38`); trust disabled → socket address,
  XFF ignored (`:73-85`).
- `req.ips` = the trusted forwarded chain, **farthest → closest**, with the socket address
  dropped: `addrs.reverse().pop()` (`:363`). Trust-all + `client, p1, p2` →
  `["client","p1","p2"]` (`test/req.ips.js:10-23`); trust disabled → `[]` (`:41-54`).

### `req.host` (`:418-431`) and `req.hostname` (`:444-458`)
- `req.host` reads `X-Forwarded-Host` **only if present and the socket is trusted**; otherwise
  falls back to the `Host` header (`:420-423`). A comma-separated trusted XFH takes the first
  value, right-trimmed (`:424-428`). Returns `undefined` if there's no host at all.
- `req.hostname` strips the port from `req.host`, with **IPv6-literal awareness**: if the host
  starts with `[`, it finds `]` first, then the `:` after it — so `[::1]:3000` → `[::1]`,
  `[::1]` → `[::1]` (`:450-457`, `test/req.hostname.js:47-71`). Untrusted or trust-disabled →
  XFH ignored, `Host` used (`:90-104,172-186`).

### `req.subdomains` (`:383-394`)
```js
var offset = this.app.get('subdomain offset');      // default 2
var subdomains = !isIP(hostname) ? hostname.split('.').reverse() : [hostname];
return subdomains.slice(offset);
```
- `tobi.ferrets.example.com` (offset 2) → `['ferrets','tobi']` (`test/req.subdomains.js:9-20`).
- **IP hosts are not split** — `127.0.0.1` → `[]` at offset 2 (`:22-33`); `[::1]` → `[]`
  (`:35-46`).
- `subdomain offset: 0` returns all reversed labels (`:96-137`).

## 8. `req.path`, `req.fresh`/`req.stale`, `req.xhr`

- **`req.path`** (`:403-405`): `parse(this).pathname` (parseurl) — the query string is stripped
  (`/login?x=1` → `/login`, `test/req.path.js:8-19`).
- **`req.fresh`** (`:469-486`): true only for a **GET or HEAD** request whose response status is
  **2xx or 304**, and whose `If-None-Match`/`If-Modified-Since` still match the response's
  `ETag`/`Last-Modified` (delegates to the `fresh` package with validators pulled from the
  *response* headers). Otherwise `false`. `If-None-Match` takes precedence over
  `If-Modified-Since` (`test/req.fresh.js:50-67`). This getter is what `res.send` consults to
  auto-emit **304** ([Chapter 6](06-the-response-object.md#3-ressendbody)).
- **`req.stale`** (`:497-499`): `!req.fresh`.
- **`req.xhr`** (`:508-511`): `(X-Requested-With || '').toLowerCase() === 'xmlhttprequest'` —
  case-insensitive; absent header → `false`.

## 9. External members you'll rely on

Not defined in `lib/request.js` but ubiquitous (grounded via tests):
- **`req.params`** — decoded route parameters; scoped and restored across routers
  (Chapter 4). `req.route` reflects the matched route pattern (`test/req.route.js:8-27` shows
  v5 `{/:op}` syntax).
- **`req.baseUrl`** / **`req.originalUrl`** — mount path / full original URL (Chapter 4).
- **`req.signedCookies`** / **`req.secret`** — from `cookie-parser`; a cookie signed with
  `res.cookie(name, val, { signed: true })` round-trips into `req.signedCookies` when the same
  secret is configured (`test/req.signedCookies.js:8-35`).

## 10. Security, concurrency & performance

- **Trust boundary:** the entire `X-Forwarded-*` surface (`protocol`, `secure`, `ip`, `ips`,
  `host`, `hostname`, `subdomains`) is client-spoofable **unless `trust proxy` is configured**.
  Default `false` ignores all forwarded headers — safe by default. When you *do* trust a proxy,
  the leftmost forwarded value (original client hop) wins on comma-separated headers.
- **Prototype pollution:** disabled `req.query` uses a null-proto object; `'extended'` opens
  `__proto__` keys via `allowPrototypes: true` (`lib/utils.js:269`).
- **Concurrency:** all members are pure getters over per-request state + immutable settings;
  no shared mutable state, no locks, fully synchronous.
- **Performance:** getters recompute each access — `req.host` re-parses on every read, and
  `req.hostname`/`req.subdomains` chain through `req.host` each time; `req.accepts*` construct a
  fresh negotiator per call (`lib/request.js:128,141,172,186`). In hot handlers, read once into
  a local variable rather than repeatedly.

## 11. Traced example: behind a trusted proxy

```js
const app = express();
app.set('trust proxy', 1);               // trust exactly one proxy hop
app.get('/whoami', (req, res) => {
  res.json({ ip: req.ip, proto: req.protocol, host: req.hostname, xhr: req.xhr });
});
```

Request from a load balancer at `10.0.0.5` forwarding a browser at `203.0.113.9`:
```
GET /whoami HTTP/1.1
Host: irrelevant.internal
X-Forwarded-For: 203.0.113.9
X-Forwarded-Proto: https
X-Forwarded-Host: app.example.com
X-Requested-With: XMLHttpRequest
```
With `trust proxy: 1`, the socket peer (the LB) is trusted for one hop, so:
- `req.ip` → `'203.0.113.9'` (the forwarded client, `lib/request.js:340-343`).
- `req.protocol` → `'https'` → `req.secure` true (`:297-315`).
- `req.host` → `'app.example.com'` (trusted XFH), `req.hostname` → `'app.example.com'` (`:418-458`).
- `req.xhr` → `true`.

With the default `trust proxy: false`, all of those forwarded values are ignored: `req.ip`
would be `10.0.0.5`, `req.protocol` `'http'`, `req.hostname` from the `Host` header.

## Where to look

- `lib/request.js` — all owned getters/methods (line refs throughout this chapter).
- `lib/utils.js:194-214` (`compileTrust`), `:162-184` (`compileQueryParser`).
- `test/req.*.js` — the executable contract for every member.

## Open questions

- Exact assignment sites for `req.params`/`req.route`/`req.baseUrl`/`req.originalUrl` are
  internal to the `router` package; behavior is pinned by `test/req.baseUrl.js`/`test/req.route.js`.
- `req.is` returning `null` and `req.range` returning `-1`/`-2` are documented but not covered
  by the in-repo tests (they'd require the `type-is`/`range-parser` sources).

**Next:** [06 · The Response Object](06-the-response-object.md).
