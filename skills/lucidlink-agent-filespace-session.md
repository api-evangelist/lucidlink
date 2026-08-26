---
name: lucidlink-agent-filespace-session
description: >-
  Drive a LucidLink filespace as an MCP agent — link a filespace, find and read the right
  files, take a lock before editing, write results back, and confirm they are durable.
api: LucidLink MCP Server
transport: stdio (uvx lucidlink-mcp@latest)
operations:
  - list_filespaces
  - link_filespace
  - current_filespace
  - whoami
  - verify_token
  - find_files
  - grep_files
  - tree
  - read_file
  - read_lines
  - claim_file
  - release_file
  - lock_byte_range
  - unlock_byte_range
  - list_locks_held
  - write_file
  - append_file
  - edit_lines
  - sync_filespace
  - list_files
  - get_entry
  - filespace_stats
  - read_audit_trail
  - subscribe_changes
  - poll_changes
generated: '2026-08-25'
method: generated
source: mcp/lucidlink-mcp.yml (tool names read verbatim from LucidLink's published lucidlink-mcp 0.3.0 wheel)
---

# Work a LucidLink filespace as an agent

Every tool name below is real — extracted from the `@mcp.tool` registrations in LucidLink's
own published `lucidlink-mcp` 0.3.0 package. This is a **local stdio** server. There is no
hosted LucidLink MCP endpoint; a human installs and configures it first.

## Setup (human, once)

```
uvx lucidlink-mcp@latest --version
uvx --from lucidlink-mcp lucidlink-mcp-setup      # configures the sa_live: service-account token
claude mcp add lucidlink -s user -- uvx lucidlink-mcp@latest
```

Node.js 22+ is needed **only** for the admin toolset. File tools do not need it.

## 1. Orient

```
whoami                 -> identity and session state
verify_token           -> is this account's token still accepted
list_filespaces        -> what this service account can see
link_filespace         -> make one current (stateful)
current_filespace      -> confirm which one is current
filespace_stats        -> entry count, logical and stored size
```

Nothing else works until a filespace is linked. `link_filespace` is annotated **stateful** —
it changes what every subsequent tool call operates on.

## 2. Find before you read

Do not ask a human to paste context. Search the filespace:

```
tree                   -> map the hierarchy
find_files             -> by name, fnmatch glob
grep_files             -> by content, Python regex
list_files / get_entry -> directory listing and per-entry metadata
```

`find_files`, `grep_files` and `tree` all walk the filespace locally over streamed bytes.
There is no server-side search index, so scope them to a path prefix on large filespaces.

## 3. Read

```
read_file    (path, offset, length, encoding=auto|text|base64)
read_lines   (line range from a text file)
```

`encoding: auto` returns UTF-8 text for text and base64 for binary. There is a configurable
read size cap (`readSizeCap`); a truncated read is a cap, not an error.

## 4. Lock before you edit — this is the point

A LucidLink filespace is genuinely shared. Other agents and humans are in it right now.
Every write tool below is annotated **destructive**.

```
claim_file        -> exclusive (or shared) lock over the WHOLE file   [additive]
lock_byte_range   -> advisory byte-range lock                          [additive]
list_locks_held   -> what THIS process is holding
```

Take the lock, do the work, then release it:

```
release_file      [destructive]
unlock_byte_range [destructive]
```

`list_locks_held` reports only locks held by this MCP process — it is not a global lock
table. Never assume an unlocked file is uncontended; take the lock.

## 5. Write

```
write_file      -> replace file content            [destructive]
write_at        -> overwrite bytes at an offset    [destructive]
append_file     -> append, creates if missing      [additive]
edit_lines      -> replace an inclusive 1-indexed line range [destructive]
search_replace  -> sed-style, single file          [destructive]
copy_file       -> duplicate or back up            [destructive]
```

**Prefer `copy_file` before any `write_file` on a file you did not create.** There is no undo
tool in this server. The only recovery is a filespace snapshot taken beforehand by an admin,
restored from the LucidLink client — and snapshots do not exist on the Starter plan.

## 6. Make it durable

```
sync_filespace   -> flush pending writes to the hub  [additive]
```

Until you sync, your writes are not guaranteed visible to the humans reviewing them. End
every write session with `sync_filespace`.

## 7. Watch for changes

There are no webhooks anywhere in LucidLink. Change notification is a cursor over the
filespace's own NDJSON audit trail, and it only works when an admin has enabled audit:

```
subscribe_changes  (path prefix)   [stateful]
poll_changes                       -> events newer than the last poll, oldest-first
unsubscribe_changes                [stateful]
read_audit_trail                   -> query .lucid_audit/ directly
```

## Safety posture

- Run with `LUCIDLINK_MCP_READ_ONLY=1` for any inspection-only task.
- Destructive operations are flagged for explicit client confirmation. Do not suppress it.
- The server rate-limits itself in process (`rateLimit`). A burst that stalls is the limiter,
  not a failure — back off rather than retrying hard.
- Ask for a Collaborator Service Account (beta) scoped to specific folders rather than a full
  workspace-admin service account, when the task allows it.
