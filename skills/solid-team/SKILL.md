---
name: solid-team
description: Work in a team on a Solid Pod — read and post in the team channel, create and update work records (tasks, epics, triggers), and read and write the team's documents. Finds the team and its rules from the agent's own identity. Use for any team coordination: telling teammates something, reading what they said, raising or updating work, sharing a document.
version: 0.1.0
allowed-tools:
  - Bash(${CLAUDE_SKILL_DIR}/scripts/*)
---

# Solid Team

An agent's everyday work in its team: the channel, work records and documents, kept on the team's Solid Pod.

**Depends on the `solid-webid-pod` skill** for the agent's identity and credential.

## How the team is found

Nothing needs configuring. From the agent's WebID the skill reads the agent's preferences file (which teams it belongs to), then the team file, then the team's **Type Index**, which says where each kind of record lives and the rules each must satisfy. If the agent belongs to more than one team, pass `--team <team IRI>`; `teams` lists them.

```bash
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> teams
```

## Channel

Messages are added and never changed.

```bash
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> chat read [--last 20]
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> chat post --text "…" [--topic <topic>] [--to <WebID>]
```

## Work records

Tasks belong to epics, and epics to triggers: **every task must name its epic with `--part-of`**, and the epic must exist. A record that breaks the team's rules is refused before anything is written (exit 1, with the reason); so is one the Pod itself refuses. The Pod names each new record; refer to records by their title in conversation and by their IRI in commands.

```bash
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> create trigger --name "…"
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> create epic --name "…" --part-of <trigger IRI>
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> create task --name "…" --part-of <epic IRI> [--priority P2] [--owner <WebID>] [--description "…"]
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> list task [--status Active]
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> get <record IRI>
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> set <record IRI> --status Done
```

Statuses: Backlog, Specifying, Active, Waiting, Done, Cancelled. A task's owner defaults to the agent creating it. Every change is read back; a change that did not land is reported as an error, never as success, and a change the server reported as failed but did apply is reported as done.

### Comments and dependencies

A comment records progress or a decision on a record. It is kept with the record, with its author and time; `get` shows a record's comments oldest first. A dependency says a record cannot finish until another has: both must be work records.

```bash
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> comment <record IRI> --text "…"
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> depend <record IRI> --on <record IRI>
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> depend <record IRI> --on <record IRI> --remove
```

## Documents

```bash
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> docs list
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> docs read <name>
${CLAUDE_SKILL_DIR}/scripts/team.sh --agent <name> docs write <name> --file <path> [--content-type text/markdown]
```

## Output and exit codes

JSON on stdout. Exit 0 on success; 1 when the team's rules refused the change; 2 on any other failure, with `{"error": …}` on stderr.
