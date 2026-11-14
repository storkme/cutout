# Working in this repo

**This repository is public, from its first commit.** History cannot be
cleaned afterwards, so nothing deployment-specific is ever committed:

- no hostnames, IP addresses or network names;
- no channel names, nicknames or anything said in a channel;
- no usage figures taken from a client's data;
- no tokens, keys or paths that reveal one;
- no security findings about a running system.

Deployment config, measurements and the design history live in the
operator's private notes, not here.

## Rules for the code

- **A key is never logged, never put in an error, never shown by `Debug`.**
- **A URL or a query from a client is never logged in the clear.** Log a
  short hash, the client, the outcome and the cost.
- **`cutout-fetch` holds no credential.** If a change needs one there, stop
  and ask.
- **The API is the owner's decision.** [SPEC.md](SPEC.md) marks what is a
  sketch. Do not settle an interface by implementing it.
- **Shared code is depended on, not copied.** A copy of fetch code that
  guards against SSRF will drift.
