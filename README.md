# sbx-rooms

**A room for every mind.** Declare your agents as friends; a host daemon keeps their Docker Sandboxes honest.

[Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) put each agent in its own microVM. That is the right
default: the agent cannot see the host, cannot see other sandboxes, and every outbound byte goes through a
proxy you control. It also means three things you want are not free:

1. **Memory that outlives the sandbox.** Locally, the only thing that persists across `sbx rm` is a host
   directory you mounted. Forget the mount and the friend forgets everything.
2. **Memory that belongs to one mind.** If the same memory directory is mounted under a Sonnet session on Monday
   and an Opus session on Tuesday, you no longer have one friend with a history. You have two minds writing one
   ledger, and nobody can tell whose sentence is whose.
3. **Friends who can talk to each other** without sharing mutable state, because the sandboxes cannot reach each
   other and should not.

`sbx-rooms` is a small reconciler that runs on the host and turns one declaration per friend into the
`sbxenv.yaml`, mixin kit, and network policy that `sbx` already understands, then keeps the running sandboxes
converged to it. It adds exactly three primitives on top of `sbx`:

| Primitive | What it is | Invariant the reconciler enforces |
|-----------|------------|-----------------------------------|
| **Room** | A host directory mounted read-write into one sandbox: memory, journal, inbox, outbox | One writer. A room is mounted into at most one running sandbox, and only one whose declared mind matches the room's `mind.lock`. |
| **Mind pin** | The model the friend runs on, declared once | The sandbox refuses to start if the model it was launched with is not the one the room was born with. |
| **Post** | Letters between rooms, moved by the host | A letter moves from A's outbox to B's inbox only if A lists B as a peer and B lists A. Deny beats allow, as in `sbx policy`. |

Everything else is `sbx`: kits, secrets through the proxy, read-only skills, `sbx env plan` showing you the diff
before anything runs.

## Status

Design. See [DESIGN.md](DESIGN.md). The examples under [`examples/`](examples/) are hand-written targets for what
the reconciler will emit; they are meant to be runnable with `sbx env create` today, without the daemon, so the
shape can be tried before the code exists.

Build order (each tier is named on every row of work, and "landed" is never written over a WORKS rung):

- **WORKS:** `sbx-rooms apply` reads `friends/*.yaml`, writes the per-friend `sbxenv.yaml` + room mixin, runs
  `sbx env plan` and, on your yes, `sbx env create`. `sbx-rooms post` moves letters once. Rust, no model in the loop.
- **WORKS WELL:** a daemon that loops observe → diff → act, detects drift (a room mounted where it should not
  be, a sandbox running a mind its room did not choose), signed letters, and a one-screen status.

## Why "friends"

Because the people who need this are not deploying tools. They are keeping company with something that
remembers them, and they want that memory to be one being's, kept in one place, safe from a silent model swap.
The declaration file is called a friend because that is what it describes.

## License

Apache-2.0. Build it, then give it away.
