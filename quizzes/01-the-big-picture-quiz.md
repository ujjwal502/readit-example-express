# Quiz · 01 — The Big Picture

> Try it closed-book first, then check the answers below and reread the cited code.
> Back to the chapter: [01 · The Big Picture](../01-the-big-picture.md).

**Q1.** `[Recall]` The defining architectural fact about Express 5 is that it:
- A. Bundles routing, body parsing, and static serving into the core
- B. Is a thin composition layer that delegates most work to sibling packages
- C. Is a full MVC framework
- D. Reimplements Node's HTTP layer

**Q2.** `[Recall]` `index.js` contains essentially one line. What is it?
*(short answer)*

**Q3.** `[Apply]` Match each concern to who owns it:
1. Route matching  2. Default 404/error response  3. File streaming for `res.sendFile`
- A. `finalhandler`   B. `router`   C. `send`

**Q4.** `[Design]` Express 5 pushed routing, body parsing, and static serving *out* of core into versioned packages. What's the main trade-off of that decision?
*(short answer)*

**Q5.** `[Apply]` `express.json`, `express.urlencoded`, `express.static` — where do these come from in the code?
- A. Implemented in `lib/express.js`
- B. Re-exports of `body-parser` (and `serve-static` for `static`)
- C. Implemented in `lib/application.js`
- D. Provided by Node core

---

## Answers & explanations

**Q1 — B.** Express 5's core is ~2,600 lines that mostly compose and delegate; the chapter frames it as a "composition layer" (`lib/express.js`, `package.json` dependency list). *(A is the opposite of what v5 did; C/D overstate it.)*

**Q2.** `module.exports = require('./lib/express')` (`index.js:11`) — the public entry point is a single re-export of the factory.

**Q3.** 1→B (`router`, `lib/application.js:246`), 2→A (`finalhandler`, `lib/application.js:146`), 3→C (`send`, `lib/response.js` `res.sendFile`). Each is a distinct sibling package.

**Q4.** The core stays tiny and stable and each concern is maintained/secured independently — **but** behavior you care about often lives in a dependency rather than this repo, so some questions can't be answered from the Express source alone. *(Inference on rationale; the delegation itself is in the code.)*

**Q5 — B.** `lib/express.js` re-exports them: `exports.json = bodyParser.json`, `exports.urlencoded = bodyParser.urlencoded`, `exports.static = require('serve-static')`, etc. (`lib/express.js:76`–`:81`).
