# cutout: spec

*Draft, 2026-11-14. Nothing is built. Sections marked "sketch" are not
agreed; the API is the owner's to decide.*

## What it is

A service that searches, fetches and reads web pages on behalf of agents.
An agent is a client. It asks; cutout does the work in a place where the
agent's keys and private network are not; text or a typed answer comes back.

It owns: the search keys, the fetching and rendering of untrusted pages, the
rules for where requests may go, the store of everything fetched, and the
model calls made on pages when a client asks for them.

Each client owns: what it asks, its own model key and budget, what it does
with an answer, and how the answer is wrapped before its own model sees it.

## Why a service

- **A browser runs the page's code.** That code makes requests from wherever
  the browser sits. Inside an agent's process that is the agent's machine and
  private network. The browser needs a place with no route to anything
  private.
- **New capabilities land once.** A renderer, PDF text, a second exit and
  prefetch are each built one time for every client.
- **One copy of the careful parts.** The address checks, the quarantined
  reader's prompt and the check of its quotes are the parts that drift when
  each agent keeps its own.
- **Clients in any language.** A client that cannot link the Rust still gets
  the search-judge-dig request.
- Search keys and egress rules in one place. A real gain, but the lesser one.

A client may keep its own in-process search and fetch and use cutout only for
what it cannot do itself.

## Two units

One binary, two units, divided by what each holds.

| | `cutout-fetch` | `cutout-think` |
|---|---|---|
| holds | no key | search keys; a model key forwarded per request |
| sees | raw bytes from strangers' pages | extracted text, search results |
| does | fetch, site fetchers, render, PDF text, archive fallback, extract, store, prefetch | search, merge, model calls, the research request |
| may reach | any public host on 80 and 443 | the search providers and the model provider |

The line is keys against hostile bytes. Nothing that parses a stranger's page
holds a key, and nothing that holds a key parses one.

## Clients and their rules

Each client has its own door (a unix socket only its account can open, or a
listener with a bearer token) and its own row of rules. A rule set says:

- **which URLs it may ask for.** Either any public URL, or only hosts on its
  allow list. A client that holds private data gets the second kind, because
  a URL it composes can carry that data out in the query string.
- **whether it may `follow`.** A client on an allow list reaches further pages
  only by number, from the links of a page cutout fetched for it. A number
  cannot carry anything out.
- **a deny list that applies to every client,** for public hosts that would
  trust the address cutout fetches from.
- its search engines, its daily allowances, and its partition of the store.

The rules live in cutout, not in the client. A client fenced so that a proxy
is its only way out would otherwise have a second way out through cutout.

## Calls (sketch)

```
POST /v1/search    { q, from?, to?, count? }
POST /v1/read      { url, question?, budget_chars? }
POST /v1/follow    { page, n }
POST /v1/verify    { page, quotes[] }
POST /v1/ask       { url, question, max_usd }            + model key
POST /v1/research  { question, schema, max_pages, max_usd } + model key
POST /v1/prefetch  { urls[] }
```

- `search` asks the client's engines together and returns one merged list.
- `read` returns a page as paragraphs, cut to the ones a question is about,
  with its links numbered, and a closed outcome word.
- `verify` says which quotes a stored page really contains.
- `ask` has a model with no tools read the page and answer, with quotes that
  are checked before they are returned.
- `research` runs a search, judges each result against the caller's schema,
  and reads further into the best, inside the ceilings.
- `prefetch` fetches now so a later `read` is warm. It returns at once.

## Fetching

A fetch picks a source by host: a site's public API where the page itself
needs JavaScript, otherwise a plain request. Every result is one of a closed
set of outcomes:

`ok` `blocked` `needs_js` `paywall` `not_found` `http_error` `unsupported`
`refused` `timeout` `error`

A bad outcome gets a second route:

- `needs_js` goes to the renderer;
- `blocked`, `paywall` and `timeout` go to another exit, then to a web
  archive;
- a PDF goes to text extraction, with a cap on size and pages.

Extraction uses no DOM. It prefers the main article over the whole body,
drops scripts and page chrome, and falls back to the page's own summary when
the body is a shell. With a question, it returns the paragraphs that share
the most words with it, plus the opening, with gaps marked.

### The renderer

- It has no route to private, link-local, loopback or carrier-grade
  addresses, for any request the page makes, not only the first.
- Its only way out is a proxy that resolves names itself, checks every
  address, and connects to the address it checked.
- It runs with its own sandbox on, and with hard caps on time and memory per
  page.
- With prefetch on, anyone who can post a link can make it run a page's code.
  The three points above are what make that acceptable.

### Exits

More than one exit is optional. Each exit is a proxy with the same address
checks, never a bare route, because an exit sits on whatever network it is
in.

## Model calls

One interface covers chat models and decision models. The first
implementation is OpenRouter. A local-model implementation is to be
considered, for a client whose rule is that hostile pages never go to a paid
third party.

- **The caller forwards its own key** with the request. cutout holds no model
  key of its own for a client's work.
- **The key should be cheap to lose.** OpenRouter lets a management key
  create an API key with a credit limit and delete it afterwards
  (`POST /api/v1/keys`, `DELETE /api/v1/keys/{hash}`; read 2026-11-14). A
  caller mints one per run with the run's ceiling as its limit, forwards it,
  and deletes it. Encrypting a long-lived key is not a substitute: cutout has
  to decrypt it to use it.
- **The provider policy travels with the request.** cutout refuses to send a
  request that lacks the client's retention policy.
- **The request carries a ceiling in dollars.** No call starts once it is
  reached. The cost counted is the cost the provider reports.
- **The full record comes back:** every model call, what it was shown, what
  it returned, its tokens and cost.

The model that reads a page has no tools and a typed output. A caller's
schema may use closed types only: booleans, numbers, choices from a list, and
quotes. A free-text field would let a hostile page write to the caller.
Quotes are checked against the stored page and dropped if the page does not
contain them.

Decision models suit the judging step, because their output is a choice or a
probability and nothing else. A decision that can drop a page before it is
read is a gate, and needs a measured miss rate before it is trusted.

## The store

Every search, raw fetch and result is kept, in the client's own partition.

- A client never sees another client's partition, or whether another client
  fetched the same URL.
- A client can delete by URL, for when a person asks to be forgotten.
- How long search results may be kept depends on each provider's terms. Not
  yet read for any of them.

## Scanning

Text that comes in is scored for injection attempts, and the score travels
with the result. Scanning changes nothing in the text. Checks for credentials
a client holds stay in that client, which is the only place that knows them.

## Logging

One line per request: the client, the call, the outcome, the cost, the time,
and a short hash of the query or URL. Never the query, the URL, or a key.
A proxy in front of `cutout-fetch` must hold to the same rule for clients
whose links are private, which rules out logging destination hosts for them.

## Where the code comes from

Most of this exists inside one of the first clients, in Rust: the search
engines and their merge, the safe fetcher and site fetchers, the extraction,
the quarantined reader, a model client and a workflow runner with a cost
ceiling. cutout depends on that code as crates. It does not copy it.

**The crates live in this repository** (decided 2026-11-14), with that client
as their first user. This repository is public, and a public project cannot
depend on a private one. Moving them here waits until that client has retired
the older code they still lean on.

New here: the renderer, PDF text, exits, prefetch, the store's partitions,
the per-client doors and rules, and the decision-model client.

## Build order

1. `cutout-fetch` with the renderer, since pages that need JavaScript are the
   largest class a client cannot read today.
2. Model calls and the `research` request.
3. The search broker, last. It adds no capability; clients search already.

Nothing starts until the code it depends on has been split out as crates.

## Open decisions

1. The API. The calls above are a sketch.
2. A licence.
3. Whether a local-model implementation is built, and which clients need it.
4. One search key per engine with an allowance per client, or each client's
   own key held by cutout.
5. How long the store keeps search results.
6. Where the renderer runs: a fenced unit beside the service, or a throwaway
   microVM per page.
7. Whether fetching with a site login is ever allowed. It changes what a
   client must tell the people whose links it reads.

## Not in scope

- A model of cutout's own that decides what to do. Code decides the steps;
  models fill them in.
- Images, audio and video.
- Publishing. Clients publish through whatever they already use.
