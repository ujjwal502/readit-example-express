# 07 · Views & Rendering

> **What you'll be able to answer after this chapter**
> - How does `res.render('index', data)` become HTML? The full path through `View` and the engine. (Control flow)
> - What is the template-engine contract, and how do EJS/Handlebars/custom engines plug in? (Interfaces)
> - How does view lookup work (roots, extensions, `index` fallback), and how is it cached? (Mechanism / State)
> - What errors can rendering produce, and what are the locals-precedence rules? (Failure / Contracts)

**Source of truth:** `lib/view.js` (205 lines), `app.render` (`lib/application.js:522-575`),
`res.render` (`lib/response.js:897-921`), `app.engine` (`lib/application.js:294-308`).
Grounded by `test/app.render.js`, `test/res.render.js`, `test/app.engine.js`, and the
`examples/` view apps (ejs, mvc, markdown, view-constructor, view-locals, route-separation).

Express ships **no template engine of its own**. It defines a small contract and adapts to
any engine that satisfies it. Its job is: resolve a template *name* to a file, pick the right
engine, call it with the merged locals, and normalize the callback.

---

## 1. The three-actor flow

```mermaid
sequenceDiagram
    participant H as handler
    participant RES as res.render (response.js:897)
    participant APP as app.render (application.js:522)
    participant V as View (view.js)
    participant E as engine (ejs/hbs/custom)
    H->>RES: res.render('users', { list })
    RES->>RES: opts._locals = res.locals; default cb = send/next(err)
    RES->>APP: app.render('users', opts, done)
    APP->>APP: merge locals; check view cache
    APP->>V: new View('users', { defaultEngine, root, engines })
    V->>V: derive ext; load engine; lookup file on disk
    alt view.path falsy
        APP-->>RES: done(Error 'Failed to lookup view "users" ...')
    else found
        APP->>V: view.render(mergedLocals, done)  (via tryRender)
        V->>E: engine(path, options, onRender)
        E-->>V: (err, html)
        V-->>APP: process.nextTick → done(err, html)
        APP-->>RES: done(err, html)
        RES->>H: send(html) or next(err)
    end
```

## 2. `res.render` → `app.render`: locals & the default callback

`res.render(view, options?, callback?)` (`lib/response.js:897-921`):
- Supports `res.render(name, cb)` (options-as-callback).
- Sets `opts._locals = res.locals` (`:911`) — this is how per-request locals reach the engine.
- If you pass **no callback**, it installs a default that **auto-responds**: `self.send(str)`
  on success, `req.next(err)` on error (`:914-917`). Passing a callback **suppresses** the
  auto-response — you get `(err, html)` and decide what to do.

`app.render(name, options?, callback)` (`lib/application.js:522-575`) does the real work:

### Locals precedence (`:536`)

```js
var renderOptions = { ...this.locals, ...opts._locals, ...opts };
```

Low → high priority: **`app.locals` < `opts._locals` (res.locals) < `opts` (render-call
locals)**. So a variable passed directly to `res.render('v', { x })` overrides the same key in
`res.locals`, which overrides `app.locals` (`test/res.render.js:251-297`,
`test/app.render.js:320-332`). `app.render` also handles `options === null` correctly
(treated as `{}`, a recent fix in `History.md`).

> **Gotcha:** `res.render` overwrites `opts._locals` with `res.locals` (`lib/response.js:911`),
> so a `_locals` you pass to `res.render` is silently dropped. Only `app.render` (called
> directly) respects a caller-provided `_locals`.

### View cache (`:538-570`)

```js
if (renderOptions.cache == null) renderOptions.cache = this.enabled('view cache');   // default from setting
if (renderOptions.cache) view = cache[name];                                          // cache HIT keyed by NAME
if (!view) { view = new View(name, {...}); if (renderOptions.cache) cache[name] = view; }
```

- Caching is controlled by the `view cache` setting (on in production, off in development by
  default, `lib/application.js:138-140`), or overridden per call via `opts.cache`.
- The cache key is the **render name**, not the resolved file path (`:545,569`) — two names
  resolving to the same file cache separately.
- With caching **off** (dev default), every render re-constructs a `View`, which re-`stat`s the
  disk (`test/app.render.js:230-288` shows the stat count incrementing when off, staying at 1
  when on).

### Lookup failure (`:558-565`)

```js
if (!view.path) {
  var dirs = Array.isArray(view.root) && view.root.length > 1
    ? 'directories "' + view.root.slice(0,-1).join('", "') + '" or "' + view.root[view.root.length-1] + '"'
    : 'directory "' + view.root + '"'
  var err = new Error('Failed to lookup view "' + name + '" in views ' + dirs);
  err.view = view;
  return done(err);
}
```

A missing template yields a precise error carrying `err.view`, with singular vs. plural
directory phrasing depending on whether `views` is an array
(`test/app.render.js:82-93,187-202`).

### Sync-throw safety (`:625-631`)

```js
function tryRender(view, options, callback) {
  try { view.render(options, callback); }
  catch (err) { callback(err); }   // a synchronous engine throw → callback, not a crash
}
```

## 3. The `View` class (`lib/view.js`)

`new View(name, { defaultEngine, root, engines })` (`:52-95`) does three things in its
constructor: derive the extension, load the engine, and resolve the file path.

### Extension derivation (`:55-73`)

```js
this.ext = extname(name);                       // e.g. '.ejs' from 'index.ejs'
if (!this.ext && !this.defaultEngine)
  throw new Error('No default engine was specified and no extension was provided.');   // :60-62
if (!this.ext) {                                 // name had no extension → use defaultEngine
  this.ext = this.defaultEngine[0] !== '.' ? '.' + this.defaultEngine : this.defaultEngine;
  fileName += this.ext;
}
```

- If the name has an extension (`'index.ejs'`), that wins.
- Otherwise the `view engine` setting (`defaultEngine`) supplies it — `res.render('index')`
  with `app.set('view engine','ejs')` → looks for `index.ejs` (`test/app.render.js:22-33`).
- With **neither** an extension nor a default engine → throws `'No default engine was
  specified and no extension was provided.'` (`test/res.render.js:39-65`).

### Engine loading (`:75-91`)

```js
if (!opts.engines[this.ext]) {
  var mod = this.ext.slice(1)                    // '.ejs' → 'ejs'
  var fn = require(mod).__express                // the engine's Express adapter
  if (typeof fn !== 'function')
    throw new Error('Module "' + mod + '" does not provide a view engine.')   // :83-85
  opts.engines[this.ext] = fn
}
this.engine = opts.engines[this.ext];
```

- If an engine is already registered for the extension (via `app.engine`), it's used.
- Otherwise Express **auto-requires** the module named after the extension and expects a
  `.__express` export (the de-facto engine convention). EJS provides `ejs.__express`, so
  `res.render('x.ejs')` works with zero configuration.
- A module without `__express` → `'Module "x" does not provide a view engine.'`
  (`test/res.render.js:39-65`).

### File resolution (`:104-187`)

```js
View.prototype.lookup = function lookup(name) {
  var roots = [].concat(this.root);              // views setting may be a string or array
  for (var i = 0; i < roots.length && !path; i++) {
    var loc = resolve(roots[i], name);
    path = this.resolve(dirname(loc), basename(loc));   // try file, then index
  }
  return path;
};
View.prototype.resolve = function resolve(dir, file) {
  var p = join(dir, file); if (tryStat(p)?.isFile()) return p;              // <path>.<ext>
  p = join(dir, basename(file, this.ext), 'index' + this.ext);             // <path>/index.<ext>
  if (tryStat(p)?.isFile()) return p;
};
```

- **Multiple roots** are tried in order; the first hit wins (`test/app.render.js:152-185`).
- For each root, it tries `<name>.<ext>` then `<name>/index.<ext>` — so a directory with an
  `index.ejs` renders as the directory name (`test/app.render.js:48-59`).
- `tryStat` swallows `fs.statSync` errors and returns `undefined` (`:197-205`), so a missing
  file just falls through to the next candidate; if nothing matches, `view.path` stays
  `undefined` → the "Failed to lookup" error.

### `View.render` — normalizing sync engines to async (`:133-159`)

```js
View.prototype.render = function render(options, callback) {
  var sync = true;
  this.engine(this.path, options, function onRender() {
    if (!sync) return callback.apply(this, arguments);        // async engine: call straight through
    var args = ...;                                            // sync engine that called back immediately:
    return process.nextTick(function () { callback.apply(cntx, args); });   // defer to next tick
  });
  sync = false;
};
```

This guarantees the render callback is **always asynchronous**, even if an engine calls back
synchronously — avoiding Zalgo (release-the-callback-sometimes-sync-sometimes-async bugs).

## 4. The engine contract & `app.engine`

**An engine is a function `(path, options, callback)`** where `callback` is
`(err, htmlString)`. Register one with `app.engine(ext, fn)` (`lib/application.js:294-308`):

```js
app.engine = function engine(ext, fn) {
  if (typeof fn !== 'function') throw new Error('callback function required');   // :295-297
  var extension = ext[0] !== '.' ? '.' + ext : ext;                             // normalize
  this.engines[extension] = fn;
  return this;
};
```

Three ways engines get wired, all from the examples:

1. **Zero-config via `.__express`** — set `view engine` and just render:
   ```js
   app.set('view engine', 'ejs');           // examples/auth/index.js:16
   app.set('views', path.join(__dirname, 'views'));
   res.render('login', { ... });            // resolves login.ejs, uses ejs.__express
   ```
2. **Map an engine to a custom extension** — register EJS under `.html`
   (`examples/ejs/index.js:23-36`):
   ```js
   app.engine('.html', require('ejs').__express);
   app.set('view engine', 'html');
   res.render('users');                     // resolves users.html, renders with EJS
   ```
3. **A fully custom engine function** — e.g. a Markdown renderer
   (`examples/markdown/index.js:17-25`):
   ```js
   app.engine('md', function (path, options, fn) {
     fs.readFile(path, 'utf8', function (err, str) {
       if (err) return fn(err);
       var html = marked.parse(str).replace(/\{([^}]+)\}/g, (m, k) => options[k] || '');
       fn(null, html);
     });
   });
   ```

`app.engine(ext, null)` throws `Error('callback function required')`
(`test/app.engine.js:32-37`); the extension normalizes with or without a leading dot
(`:39-51`).

### Replacing the `View` class entirely

The `view` setting is the View **constructor**; you can swap it to change resolution wholesale
(`examples/view-constructor/index.js:30` sets `app.set('view', GithubView)`). A custom view
must implement `constructor(name, options)` (reading `options.engines`, `options.root`), expose
a truthy `.path` on success, and a `.render(options, cb)` method — that's the entire contract
`app.render` depends on (`test/app.render.js:206-227`).

## 5. Per-controller engines (the MVC example)

`examples/mvc` shows multiple engines coexisting. Its `boot.js` reads each controller's
`exports.engine` and does `app.set('view engine', obj.engine)` per sub-app — the `user`
controller uses `hbs` (Handlebars, `.hbs` templates with `{{#each}}`), the `pet` controller
uses `ejs` (`examples/mvc/controllers/user/index.js:9`, `pet/index.js:9`,
`mvc/boot.js:27`). Because engines are stored per app in `this.engines` and inherited on mount
(`lib/application.js:120`), each mounted controller app can register and use a different engine.

## 6. Errors, edge cases & gotchas

| Situation | Result | Grounded |
|---|---|---|
| No extension and no `view engine` | `Error('No default engine … no extension …')` | `test/res.render.js:39-65` |
| Extension's module lacks `__express` | `Error('Module "x" does not provide a view engine.')` | `test/res.render.js` |
| Template file not found | `Error('Failed to lookup view "…" in views …')` with `err.view` | `test/app.render.js:82-93,187-202` |
| Engine calls back with an error | propagated to the render callback → `next(err)` (default) | `test/res.render.js:112-130` |
| Engine throws synchronously | caught by `tryRender` → callback | `test/app.render.js:61-80` |
| `app.engine(ext, non-fn)` | `Error('callback function required')` | `test/app.engine.js:32-37` |
| `_locals` passed to `res.render` | **dropped** (overwritten by `res.locals`) | `lib/response.js:911` |
| `view cache` off (dev default) | re-stat disk every render | `test/app.render.js:230-258` |
| `views` default | `<process.cwd()>/views`, fixed at `express()` time | `lib/application.js:135` |

## 7. Traced example: `res.render('users', { list })` with EJS

```js
app.set('view engine', 'ejs');
app.set('views', __dirname + '/views');
app.locals.title = 'My App';
app.get('/users', (req, res) => {
  res.locals.now = Date.now();
  res.render('users', { list: ['a','b'] });
});
```

1. `res.render('users', { list })` → sets `opts._locals = res.locals` (`{ now }`), installs the
   default send/next callback, calls `app.render('users', opts, done)`.
2. `app.render` merges locals: `{ title:'My App', settings, now, list:['a','b'] }` (app.locals
   < res.locals < opts). `view cache` is off in dev → build a fresh View.
3. `new View('users', { defaultEngine:'ejs', root:'.../views', engines })`: no extension in
   `'users'` → ext becomes `.ejs`, filename `users.ejs`. Engine: `engines['.ejs']` is empty →
   `require('ejs').__express`. Lookup: `.../views/users.ejs` exists → `view.path` set.
4. `tryRender` → `view.render(mergedLocals, done)` → `ejs.__express('.../users.ejs',
   mergedLocals, onRender)`. EJS reads/compiles/renders the template with `list`/`title`.
5. On success the (nextTick-deferred) `done(null, html)` fires; the default callback calls
   `res.send(html)` → `200`, `text/html; charset=utf-8`, body = rendered HTML.

If `users.ejs` didn't exist, step 3 leaves `view.path` undefined → `app.render` calls
`done(Error('Failed to lookup view "users" in views directory ".../views"'))` → the default
callback calls `req.next(err)` → finalhandler → **500**.

## Where to look

- `lib/view.js` — the `View` class: constructor, `lookup`, `resolve`, `render`, `tryStat`.
- `lib/application.js:522-575` (`app.render`), `:294-308` (`app.engine`), `:134-140` (view
  defaults).
- `lib/response.js:897-921` (`res.render`).
- `examples/ejs`, `examples/markdown`, `examples/mvc`, `examples/view-constructor` — engine
  wiring recipes. `test/app.render.js`, `test/res.render.js`, `test/app.engine.js` — the spec.

## Open questions

- The internals of specific engines (EJS/Handlebars) are third-party; Express only depends on
  the `(path, options, callback)` / `.__express` contract described above.

**Next:** [08 · Bundled Middleware](08-bundled-middleware.md).
