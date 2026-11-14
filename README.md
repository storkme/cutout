# cutout

A go-between for agents that read the web. An agent asks cutout to search,
fetch a page, or answer a question about one. cutout does the fetching and
the reading somewhere the agent's keys and private network are not, and hands
back text or a typed answer.

In tradecraft, a cutout is the intermediary who lets two parties deal without
ever touching each other. Here the two parties are an agent and a stranger's
web page.

**Status: a design, nothing built.** The design is [SPEC.md](SPEC.md).

## What it is for

Mostly new capabilities, for agents that already search and fetch on their
own:

- pages that are empty without JavaScript, read through a browser that has no
  route to anything private;
- one request that searches, judges each result against a schema, and reads
  further into the best ones, under a ceiling in pages and dollars;
- prefetch, so a page is kept as it was the day it was linked;
- every search and fetch kept, so runs can be replayed in tests.

It also puts search keys and egress rules in one place, but that is the
lesser reason.

## Shape

One Rust binary, run as two units:

| unit | holds | touches |
|---|---|---|
| `cutout-fetch` | no key | strangers' pages: fetch, render, extract, store |
| `cutout-think` | search keys, and a model key the caller forwards | extracted text only: search, model calls |

A key never sits in the process that parses a stranger's page.

## This repository is public

Nothing about a particular deployment belongs here: no hostnames, addresses,
channel names, usage figures or tokens. A deployment's config lives with
whoever runs it.
