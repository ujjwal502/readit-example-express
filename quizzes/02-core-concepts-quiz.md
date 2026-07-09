# Quiz · 02 — Core Concepts

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [02 · Core Concepts](../02-core-concepts.md).

**Q1.** `[Recall]` What is the prototype chain of a live request object during handling?
- A. `req` → `IncomingMessage` only
- B. `req` → `app.request` → `lib/request.js` → `http.IncomingMessage.prototype`
- C. `req` → `Object.prototype`
- D. `req` → `lib/response.js` → `ServerResponse`

**Q2.** `[Apply]` `app.set('etag', 'strong')` stores `settings['etag'] = 'strong'`. What *else* happens as a side effect, and where?
*(short answer)*

**Q3.** `[Recall]` Why is `app` mixed with `EventEmitter.prototype` at construction?
- A. So `app.listen` can emit `'listening'`
- B. Mainly to support the `'mount'` event when sub-apps are attached
- C. To stream response bodies
- D. It's required by `http.createServer`

**Q4.** `[Apply]` A sub-app is mounted with `app.use('/admin', subApp)`. After the sub-app finishes handling and calls `next`, what must happen to `req`/`res` before control returns to the parent?
*(short answer)*

**Q5.** `[Design]` `app.request` and `app.response` each carry an `app` back-reference. Why is that needed?
*(short answer)*

---

## Answers & explanations

**Q1 — B.** Express prototypes inherit from Node's (`lib/request.js:29`), and per request `app.handle` does `Object.setPrototypeOf(req, this.request)` (`lib/application.js:163`), where `this.request` → `app.request` → `lib/request.js` exports → `IncomingMessage.prototype`.

**Q2.** `app.set` special-cases `etag`: it also compiles and stores `settings['etag fn'] = compileETag(val)` (`lib/application.js:298`). That compiled function is what `res.send` later calls to generate ETags. Three keys are reactive this way: `etag`, `query parser`, `trust proxy`.

**Q3 — B.** The `EventEmitter` mixin (`lib/express.js:40`) exists mainly so a mounted sub-app can `emit('mount', parent)` and the parent/child can wire up prototype inheritance (`lib/application.js:104`, `:223`). *(Inference on the primary motivation.)*

**Q4.** The wrapper restores the parent's prototypes: `Object.setPrototypeOf(req, orig.request)` / `...(res, orig.response)` before calling the parent's `next` (`lib/application.js:216`) — otherwise the parent would keep operating on `req`/`res` still pointing at the sub-app's prototypes.

**Q5.** So that `req.app` / `res.app` resolve to the owning application. The back-ref is set when `app.request`/`app.response` are created via `Object.create(req, { app: { value: app } })` (`lib/express.js:44`), letting request/response helpers read app settings (e.g. `this.app.get('trust proxy fn')`).
