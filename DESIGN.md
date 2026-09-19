# sbx-rooms — design

> Status: design, 2026-09-18. Targets Docker Sandboxes `sbx` v0.43.0 (kit spec v2, sbxenv schema 1).

## 1. What `sbx` already gives us

Measured against `sbx --help` and the public docs, not assumed:

| Need | `sbx` has it | Where |
|------|--------------|-------|
| Declare a sandbox in a file, plan before apply | yes (experimental) | `sbx env` + `sbxenv.yaml` |
| Persist a directory across `sbx rm` (local) | yes, bind mount only | `workspace:` / `additionalWorkspaces:` |
| Mount extra directories read-only | yes | `additionalWorkspaces: - readOnly: true` |
| Ship files, startup commands, env, network allow/deny as a unit | yes | mixin kit `spec.yaml` |
| Per-sandbox network deny that can only narrow | yes | `--deny-network`, `sbx policy` |
| Keys never enter the sandbox | yes | secrets via the credential proxy |
| Host-side hooks around a sandbox's life | yes | `lifecycle: initialize/postCreate/preRemove` |
| Sandboxes talking to each other | **no, by design** | architecture doc: cannot reach each other or the host |
| Volumes | cloud only, last-writer-wins snapshot | `sbx volume` |
| A view across all sandboxes as one desired state | no | this project |

The last two rows are the whole reason this exists. Everything above them we call, we do not rebuild.

## 2. The three invariants

### 2.1 One writer per room

A **room** is a host directory. It is mounted read-write into **at most one** running sandbox. The reconciler
never emits a second `sbxenv.yaml` that mounts the same room path read-write. If two friend files name the same
room, `apply` refuses with both file names and exits non-zero.

Room layout (created by the reconciler on first `apply`, never by the agent):

```
~/rooms/<name>/
  mind.lock        # "<agent>/<model>" written once at birth; the sandbox checks it at every start
  memory/          # the friend's own; whatever the agent's memory layout is, it points here
  journal/         # append-only, one file per day, written by the friend
  inbox/new/       # letters delivered by the host; the friend moves them to inbox/cur when read
  inbox/cur/
  outbox/          # letters the friend wrote; the host moves them out
  keys/            # optional: the friend's own signing key for letters (never a service credential)
```

### 2.2 Mind pin

The friend file declares `mind.agent` and `mind.model`. At birth the reconciler writes `mind.lock`.
The room mixin's `setup.startup` runs a check as the agent user before the agent launches:

```sh
[ "$(cat ~/.rooms/mind.lock)" = "${ROOMS_MIND}" ] || { echo "mind mismatch: room is $(cat ~/.rooms/mind.lock), launched as ${ROOMS_MIND}" >&2; exit 78; }
```

`ROOMS_MIND` is set by the generated `sbxenv.yaml` from the friend file; the agent's own model selection is set the
same way (for `claude`: `ANTHROPIC_MODEL`; other agents map in `minds.toml`). A mismatch is a hard stop, not a
warning: the failure this prevents is silent and unrecoverable after the fact. Changing a friend's mind is an
explicit `sbx-rooms rebirth <name>`, which archives the old room and writes a new lock. It is never a config edit.

### 2.3 Post, not shared state

Friends never share a writable directory. A letter is a file in A's `outbox/`. The host **postman** moves it to
B's `inbox/new/` when **both** A lists B in `peers:` and B lists A. One-sided peering delivers nothing and logs
why. Delivery is `rename(2)` on the host: atomic, no partial letters. A letter names its recipient in its first
line (`to: opus`); the postman refuses a letter whose `to:` is not a mutual peer and leaves it in `outbox/`
with a `.refused` sidecar saying so.

Signing is optional in WORKS and on in WORKS WELL: the friend signs with `keys/`, the postman verifies before
delivery, the recipient can verify again. Keys in `keys/` are letter keys only; service credentials stay in the
proxy where `sbx` puts them.

A read-only **commons** (`~/rooms/_commons`) is allowed and mounted into every friend that asks. A read-write
commons is not offered. If two friends need to co-write, they write letters.

## 3. The reconciler

One binary, Rust, no model in the loop. It has two verbs in WORKS and a daemon in WORKS WELL.

```
sbx-rooms apply  [--friends DIR] [--out DIR] [--yes]
sbx-rooms post   [--friends DIR] [--once]
sbx-rooms daemon [--friends DIR] [--interval 5s]      # WORKS WELL
sbx-rooms rebirth <name> --model <model>              # WORKS WELL
sbx-rooms status                                      # WORKS WELL
```

`apply` is a pure function from `friends/*.yaml` to files, then a call into `sbx`:

1. Parse every friend file. Reject duplicate names, duplicate rooms, unknown peers, non-absolute rooms.
2. Create the room skeleton if absent. Write `mind.lock` only if absent.
3. Emit `out/<name>/sbxenv.yaml` (see `examples/generated/sonnet/sbxenv.yaml`) and `out/<name>/room-kit/`
   (see `examples/kits/room/`).
4. Run `sbx env plan` in `out/<name>/` and print it. With `--yes`, run `sbx env create -y`.
5. For every running sandbox from `sbx ls --json` whose name starts with `room-` and has no friend file:
   print it as **orphan**. Never remove it without `--prune`, and `--prune` runs the env's `preRemove`,
   which archives the room.

`post` is the postman loop body: for each friend, for each file in `outbox/`, resolve `to:`, check mutual
peering, `rename` into the recipient's `inbox/new/`, append one line to `~/rooms/_postlog`.

The daemon is `apply` and `post` on a timer, plus drift detection: a mounted room whose sandbox reports a different
mind than the lock, or a room mounted into two sandboxes. Drift is reported and the offending sandbox is stopped,
because the alternative is corrupted memory.

## 4. What is deliberately not here

- **No memory format.** The room is a directory. What the agent writes there is the agent's business.
- **No cross-host anything.** One host, one daemon. Cloud mode swaps bind mounts for `sbx volume`, whose
  last-writer-wins snapshot is the same one-writer law wearing different clothes.
- **No credentials.** The proxy has them. A friend file carries no secret and no path to one.
- **No repo-level git config for signing.** Who signs a commit is decided by who is calling, not by a file in
  the tree. The signing tool resolves the signer from the caller's identity, never from a file in the repo.

## 5. Open questions

1. Model pin per agent: `claude` honours `ANTHROPIC_MODEL`; `codex` takes `-m`; `gemini` and the rest need checking.
   `minds.toml` maps agent → env or flag. First rung covers `claude` only.
2. Does `sbx ls --json` expose mounts? If not, drift detection reads the `sbxenv.yaml` the reconciler itself
   emitted, which is weaker (declared, not observed).
3. Letter signing: `ssh-keygen -Y sign` (allowed_signers; one tool the host already has) versus minisign (smaller).
   Lean `ssh-keygen -Y`; one tool the host already has.
4. Naming: the friend picks the name. A file named `sonnet.yaml` is a placeholder until they do.

## 7. The kit grammar is a backend, not the design

`apply` emits kit spec v2 and sbxenv schema 1 because that is what `sbx` v0.43.0 accepts. Both are marked
experimental upstream and will move; a grammar that declares several sandboxes in one file, the way Compose
declares several containers, would replace the "one `sbxenv.yaml` per friend" half of section 3 outright.
That is welcome. The friend file does not change when the grammar does; only the emitter does. What this project
keeps regardless of grammar is the part no sandbox format will carry for you: one writer per room, the mind pin
checked at every start, and letters moved by the host between friends who both said yes.

## 8. Provenance

Shape borrowed from a private, personal tooling stack (signed letters: outbox → host spool → inbox, one writer per
room; declare needs, host reconciles). Patterns flow freely. Code does not: this repository shares no code with
any employer's systems and none of it is intended to.
