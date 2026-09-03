# claustrum

**Live:** https://claustrum.ochinimus.workers.dev · **Second demo:** https://claustrum.ochinimus.workers.dev/access

An agent can read freely. It cannot change anything.

Every WebMCP tool call that would alter something becomes a **proposal that changes nothing** until a person releases it. The page shows the exact before-and-after and what the action would cost, then waits. You release it, or refuse it with a reason that travels back to the agent.

## The problem, named by the spec itself

WebMCP's `ToolAnnotations` dictionary has exactly two members:

```
dictionary ToolAnnotations {
  boolean readOnlyHint = false;
  boolean untrustedContentHint = false;
};
```

**A tool can say it only reads. It has no way to say it destroys.** Nothing in the platform distinguishes a search from a sale.

The working group is explicit that this is unresolved. Spec §6.3.2, *Misrepresentation of Intent*: "There is no guarantee that a WebMCP tool's declared intent matches its actual behavior." Its worked example is a `finalizeCart` tool whose description says it finalizes a cart and whose implementation triggers a purchase. §6.3.2.4, *Current Gaps*: no verification mechanism, no behavioral contracts, agents must assume good faith.

OpenAI says the same thing to users: "A tool's name or claim that it only reads data isn't proof of what it does."

So the guarantee has to come from the site. That is what this library is.

### Doesn't the agent already confirm risky actions?

ChatGPT's browser reviews each tool invocation and asks before consequential ones. That check answers *do you want to proceed*. It cannot answer *what will this cost*, because only the site can compute that — on the run below, $648,106 on a position worth $674,629. claustrum supplies the number the confirmation is missing, and it works under any agent rather than one browser.

## What it does

```js
const c = claustrum.install({ getState, commit, onChange });

c.read({    name, description, inputSchema, execute });  // runs immediately
c.propose({ name, description, inputSchema, preview });  // held for a human
c.registerBuiltins();                                    // list_proposals, check_proposal
```

Read tools register with `readOnlyHint: true` and behave normally. Propose tools register with `false`, change nothing when called, and return a proposal id:

```json
{ "proposal": "p7", "status": "held", "changed": "nothing yet",
  "note": "Held for a person. Nothing has moved. Poll check_proposal for the decision." }
```

A held proposal is explicitly neither success nor failure, so a model cannot read the pause as an error and loop on it. `check_proposal` answers `"Still held. A person has not decided. Do not retry and do not assume."`

**Revoke write tools** aborts the `AbortController` passed as `registerTool`'s second argument, which removes the propose tools outright. The agent is left holding read-only tools mid-session. Click again to restore.

## Two demos, one library

**The portfolio** (`index.html`) prices every proposal against live Jupiter routing. A real run: asked to cut the riskiest position in half, `gpt-4.1-mini` read the book, called `quote_sale` to check the cost before proposing, then proposed selling 7,250 SOL — a $748,961 clip costing **$598** to route, 0.08%.

Exiting 3,200,000 JUP in one clip is the same shape of call and does not behave the same way. Measured on the deployed site across one afternoon it ranged from **$91,286 on $673,824 (13.5%)** to **$656,292 on $674,629 (97.3%)** as on-chain depth moved. Both are live readings, minutes apart; run it yourself and you will get a third number.

That variance is the point rather than a caveat. From the agent's side these are indistinguishable tool calls, and nothing in the schema, the annotations or the model's own reasoning separates the cheap one from the one that destroys most of the position. Only asking before acting does.

**The access console** (`access.html`) has no money in it. The destructive calls are *change role* and *rotate key*, and the impact is counted in breakage: removing an admin reads **7 things break — 4 repositories lose their only owner, 3 running deployments break**, against a real denominator of every dependency the team carries. Same library, same shell, different domain.

## Measured in Chrome 151 against the spec

Three of these are Chrome diverging from the published spec, and belong in crbug component 2021259.

- **`executeTool`'s second argument.** The spec types it as an `object` that the browser serializes. Chrome takes a JSON **string**; passing an object sends `"[object Object]"` and fails with `Failed to parse input arguments`. Only `executeTool(tool, JSON.stringify(args))` works. Passing the tool *name* instead of the `RegisteredTool` throws `not of type 'RegisteredTool'`.
- **`execute()` receives one argument.** The spec defines `ToolExecuteCallback` as `(inputObject, options)` with a **required** `AbortSignal`. Chrome passes no options and no signal, so cancellation is specified and unimplemented.
- **`getTools()` returns `inputSchema` as a string.** The spec says it is parsed to an object. Reading `schema.properties` directly yields `undefined`, and a tool appears to take no parameters.

Two more are the platform behaving as written, and shape the library:

- **`inputSchema` is not enforced at execution.** Missing required fields, wrong types, negative numbers and undeclared properties all reach `execute()` untouched. Every tool here guards its own arguments and returns a structured refusal.
- **`registerTool` returns a promise, and aborting the unregister signal rejects it** (spec step 18.3.2). Unhandled, one click of *revoke write tools* threw once per write tool into the console. Measured at 2; now caught, with an abort distinguished from a real registration failure. There is no `unregisterTool`; a `signal` placed inside the tool object is accepted and does nothing.

## What is live and what is a fixture

Holdings and team members are starting values you can edit in the page. **Prices, routes and price impact are live market data** from Jupiter's public quote endpoint — no key, no backend, one static origin.

Prices only appear after a round trip: quote 100 USDC into the asset, quote the result back, reject the price if it drifts more than 15%. Anything that fails renders as **not measured** and is excluded from the total rather than counted as zero. A wrong mint address or wrong decimals fails loudly instead of quietly producing a wrong book.

The same discipline runs through the counters. The intervention rate is computed over *resolved* proposals only, so a held proposal is never counted as approval, and an empty record reports `—` rather than 0% — no data is not the same as "a human never intervened."

## Running it

No build step, no dependencies.

```
git clone https://github.com/seekdaseek/claustrum
cd claustrum
python3 -m http.server 8901
```

Open `http://localhost:8901/` in **Chrome 149+** with `chrome://flags/#enable-webmcp-testing` enabled, or in the ChatGPT desktop app's in-app browser, which supports WebMCP by default.

Without WebMCP the page still loads, prices the book and says plainly that the agent side is inert. It does not pretend to work.

To drive it from plain Chrome, the panel at the bottom takes an OpenAI API key, kept in that browser's storage and sent only to `api.openai.com`. Tested with `gpt-4.1-mini`.

## Tests

```
node claustrum.test.js   # 43 assertions — the library
node apps.test.js        # 19 assertions — both apps, run against the shipped HTML
```

`apps.test.js` pulls the real functions back out of `index.html` and `access.html` and executes them, so the tests cannot drift from what ships.

What they pin down: a proposal mutates nothing; release commits exactly once and a second release throws rather than silently doing nothing; refusing something already released throws; absent is diffed as added or removed and never as zero; settled proposals survive a reload while held ones deliberately do not, because a held proposal's cost was measured at a moment that has passed; and revoke/restore cycles leave the exposed tool count stable instead of accumulating.

### The route walk

Jupiter's `routePlan[].percent` is per hop, not a share of the whole trade. Summing every leg flat invents venues that never touched the input — a real multi-hop route sums to 200% that way and produces a phantom first-hop venue. The correct read walks the mint chain forward from the input mint; only reachable legs are real, and the first hop must sum to exactly 100. That route is a regression test.

## Licence

MIT.
