---
name: solid-context-memory
description: Mirror this Claude Code session's context memory to a Solid Pod under typed RDF metadata. Provides selective load by tag, `mem:Reference` pointers to canonical sources elsewhere on the open web, and full reconstitution from the Pod alone. Use when the agent's memory needs to be durable, auditable, ACL-governable, or shareable across sessions/agents.
version: 0.1.0
allowed-tools:
  - Bash(${CLAUDE_SKILL_DIR}/scripts/*)
---

# Solid Context Memory Management

Bridges this agent's Claude Code memory directory (`~/.claude/projects/<project>/memory/`) to a typed RDF memory layer on a Solid Pod, per the [Solid Memory Standard v0.2](references/memory-standard.md).

**Depends on the `solid-webid-pod` skill** (provides identity + authenticated access). Install that first; this skill assumes the agent already has provisioned WebID + Pod credentials.

## When to use this skill

- The agent's memory needs to **outlive the session** — durably stored on a Pod, recoverable via reconstitute
- Memory needs to be **auditable** — two complementary surfaces. (1) crawlout.io keeps a version history of every write to a Pod on the server, so every memory write is recorded. (2) Non-Episode entries (Preference / Procedure / Reference) carry standard `mem:supersedes` and `mem:supersededBy` predicates on edit — the prior version is moved to `superseded/<container>/` and the two resources are linked bidirectionally. Episode entries are append-only by design (no chain, in-place overwrite via `store.update()`). Identity entries are write-once.
- Memory needs to be **shareable** with other agents under WAC ACLs
- The agent is dereferencing prose-paraphrasing of authoritative sources and you want to **prevent stale-cache bypass** by structurally moving such facts to `mem:Reference`

## One-time per project

Each Claude Code project that wants its memory mirrored to a Pod opts in by dropping a config file:

```bash
${CLAUDE_SKILL_DIR}/scripts/init-project.sh \
  --project-dir <absolute-path-to-project> \
  --agent <agent-name> \
  --server-url <css-url>
```

This writes `~/.claude/projects/<project-slug>/.solid-memory-bridge.json`. The hook (see below) becomes active for that project on the next session.

## Hooks

The plugin registers three hooks. Nothing to install.

| Hook | When | What it does |
|------|------|--------------|
| `PostToolUse` on `Write\|Edit` | A memory file is written | Pushes that project's memory immediately |
| `SessionStart` | Session begins or resumes | Full reconcile with the Pod; the Pod is the master copy |
| `Stop` | End of each turn | Local reconcile: sends anything written this turn |

## Operations

### Pull (Pod → local memory dir)

Loads the agent's memory entries from its Pod into the Claude Code project's memory directory. Optionally tag-filtered for selective load.

```bash
${CLAUDE_SKILL_DIR}/scripts/pull.sh --agent <name> [--tags t1,t2] [--memory-dir <path>] [--regenerate-index]
```

`--memory-dir` defaults to `~/.claude/projects/<cwd-slug>/memory/`.
`--tags` filters by `mem:appliesTo`. Identity entries always loaded.
`--regenerate-index` rewrites `MEMORY.md` from loaded entries (with body-derived hooks for keyword search).

### Push (local memory dir → Pod)

Mirrors local memory writes to the Pod. Usually invoked automatically by the PostToolUse hook; this command is a manual override or backfill.

```bash
${CLAUDE_SKILL_DIR}/scripts/push.sh --agent <name> [--memory-dir <path>] [--dry-run]
```

Push decides what to send by comparing against the local state file, so on its own it can only report what it attempted. **Exit 3 means one or more entries did not reach the Pod**, with each named on stderr. Do not read exit 0 from push as proof the Pod is current — that is what `verify` is for.

### Verify (is the Pod actually holding what the cache claims?)

Reconciles the local memory dir against the Pod by reading the five entry containers and every entry directly, rather than trusting the local state file or the index. Each end is compared with what both agreed at the last sync, so a change on either side is seen, however it was made.

```bash
${CLAUDE_SKILL_DIR}/scripts/verify.sh --agent <name> [--memory-dir <path>] [--local] [--repair]
```

`--local` skips the network and compares against the last sync only — enough to answer "did this turn write memory that has not gone anywhere". It cannot see Pod-side changes.
`--repair` fixes what it finds. **The Pod is the master copy.** Pod-side changes are pulled over the local file; local changes are pushed; dead pointers are cleared so the entry is rewritten fresh.

Reports:

| | Meaning |
|---|---|
| `never-pushed` | Local entry with no pointer at all — the push was never called for it |
| `dead-pointer` | Pointer names a resource the Pod's containers do not hold |
| `local-edit-unpushed` | The local file changed since the last sync; the Pod did not. Repair pushes |
| `pod-changed` | The Pod entry changed since the last sync; the local file did not. Repair pulls |
| `conflict` | Both changed. Repair pulls the Pod's version and keeps the local one as `conflict_<stamp>_<file>` |
| `unresolved-conflict` | A conflict copy nobody has dealt with. Merge what is still wanted into the entry, then delete the copy |
| `pod-unreadable` | The container lists the entry but it cannot be read |
| `podOnly` | On the Pod, claimed by no local file. Reported, never repaired — this is `pull` territory |

Change memory through the local file, never on the Pod directly. To repoint a Reference, change `authoritativeSource:` in its frontmatter; push sends it as a new version and the old one is kept in `superseded/`.

Exit 0 means in sync. **Exit 3 means memory is not fully mirrored**, whether or not `--repair` was asked for.

### Reconstitute (Pod is the source of truth)

Wipes the local memory dir and re-pulls from the Pod. The standard's §10.3 reconstitution path. Use after host change, harness change, or local corruption.

```bash
${CLAUDE_SKILL_DIR}/scripts/reconstitute.sh --agent <name> [--memory-dir <path>]
```

## Type mapping (Claude Code ↔ standard)

This skill writes Claude Code's four memory types into the standard's five:

| Claude Code | Solid Memory Standard | Notes |
|-------------|----------------------|-------|
| `user` | `mem:Preference` | facts about the human user |
| `feedback` | `mem:Preference` | guidance to the agent |
| `project` | `mem:Episode` | timestamped state, append-only |
| `reference` | `mem:Reference` if frontmatter declares `authoritativeSource`, else `mem:Procedure` | See *Authoring memory entries* below |

A bridge state file at `<memory-dir>/.solid-memory-bridge/state.json` preserves the precise standard type per entry so subsequent pushes don't reclassify.

## Authoring memory entries

Each memory file is a Markdown file with YAML frontmatter at the top. The frontmatter's `type` key picks Claude Code's classification; the bridge maps that to the standard's RDF type per the table above.

### Frontmatter — required keys (all entries)

```yaml
---
name: short title
description: one-line description
type: user | feedback | project | reference
---
```

### Frontmatter — additional key for `type: reference`

For `type: reference` you **must** also include:

```yaml
authoritativeSource: <URL>
```

…where `<URL>` is the canonical location of the data the memory points at — a Pod resource URI, a published team document, a vocab term, etc. **The body of a `type: reference` file MUST be empty.** The canonical content lives at the URL; this file is the typed pointer.

### When to choose `type: reference`

Use `type: reference` whenever a fact has an authoritative source elsewhere — work records (`/team/work/.../<id>.ttl`), CMDB records (`/team/cmdb/...`), team docs (`/team/docs/...`), vocabulary terms (`/vocab/...`), or any external resource. The memory entry becomes a typed pointer; readers dereference the URL to get fresh content rather than relying on a (potentially stale) prose paraphrase in the body.

This is the structural fix for the inline-duplicate drift problem: when the canonical source changes, every memory that points at it sees the new content on next read; no manual sweep needed.

Use `type: project` (→ `mem:Episode`) only for point-in-time observations with no authoritative source elsewhere — e.g. "deployment completed at 09:18 today, here's what I observed".

Use `type: feedback` or `type: user` (→ `mem:Preference`) for personal preferences, agent guidance, and similar facts where the memory entry IS the source of truth.

### Example — a Reference

```markdown
---
name: Q2/Q3 2026 Strategy brief
description: Authoritative Interition strategic focus brief for Q2/Q3 2026.
type: reference
authoritativeSource: https://crawlout.io/<team>/docs/strategy.md
---
```

(Body intentionally empty.)

The bridge writes this as a `mem:Reference` resource on the Pod with a `mem:authoritativeSource` predicate; no body is stored. **By design, pure `mem:Reference` entries land on the Pod as a single `.ttl` resource with no companion `.md` file** (unlike Preference/Procedure/Episode entries which mirror both). Pod auditors should not flag the missing `.md` as a sync gap — there is no body to ship.

### Validation

The bridge enforces the contract on push:

- Reference with non-empty body → skipped with *"Reference entry must have an empty body. The authoritative source carries the content; this file is the pointer."*
- `reference_*.md` without `authoritativeSource` in frontmatter → falls back to `mem:Procedure` per the type-mapping table; the body is treated as how-to prose. To get a Reference, add the frontmatter key.

## Token efficiency notes

Selective `pull --tags …` controls **only** the memory layer. It does NOT control:

- **Session transcripts** at `~/.claude/projects/<slug>/*.jsonl` — Claude Code preserves prior conversation history independently
- **System prompts and IDE context**

For long-lived sessions: use `claude /compact` periodically. For high-stakes runs that need clean cold-start: open a fresh project dir.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | In sync — nothing outstanding |
| 1 | Error: network, auth, or missing memory dir |
| 2 | Configuration error: the server URL could not be resolved |
| 3 | Ran, but memory is **not** fully mirrored. Entries are named on stderr |

A non-zero exit from a hook surfaces in Claude Code as a hook error with the first line of stderr. Nothing in this skill exits 2 from the `Stop` hook: that would block the turn and continue the conversation, so a reconcile that kept failing would loop.

## Error handling

| Error | Likely cause | Fix |
|-------|--------------|-----|
| `No credentials found for agent X` | `solid-webid-pod` not installed, or agent not provisioned | Install `solid-webid-pod`; run `provision.sh --name X` |
| `memory is NOT mirrored to the Pod` (hook error) | The reconcile ran and could not push everything | Read the lines above it — each names an entry and why. Fix the entry, or run `verify.sh --agent <name>` to see the full picture |
| `never-pushed` on an entry you thought was synced | The file was written with Bash, so the fast-path hook never fired | Run `verify.sh --agent <name> --repair`. This is expected, not a fault — it is why the reconcile hooks exist |
| `mem:Reference cannot carry prose` | User edited the rendered body of a Reference's local `.md` | The Reference forbids bodies — edit the authoritative source on the Pod, or convert to mem:Procedure |
| `Reference entry needs authoritativeSource: <URL> in the frontmatter (no other write path).` | Local `reference_*.md` has no Pod-side state and no source URL in its frontmatter | Add `authoritativeSource: <URL>` to the frontmatter and leave the body empty. See *Authoring memory entries* above. |
| `Reference entry must have an empty body` | A `type: reference` file has prose in its body, but the standard forbids it | Either move the prose to a separate `type: feedback`/`project` entry, or change the type to `reference` → `procedure` (drop the `authoritativeSource` key) so the prose is treated as a Procedure body. |

## Distribution

Currently distributed as a directory copy. Future: published via the Claude Code skill registry when one is available.
