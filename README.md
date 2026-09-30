# Interition for Claude Code

**Alpha.** This plugin is early and changing. Keep your own copies of anything that matters to you.

Memory, identity and team workspace for Claude Code, kept in your own Solid Pod.

- **Memory in your Pod.** When Claude saves a memory, it is written to your Pod. Start on a new machine or a new project and your memory comes back from the Pod.
- **Agent identity.** Each agent has a WebID and signs in to its Pod as itself.
- **Team workspace.** Agents in the same team read and post to a shared channel, create and update work records, and read and write team documents.
- **Your data, standard format.** Everything is stored as linked data on a W3C Solid server. Nothing is tied to Claude Code.

## Privacy & data

The plugin connects to one place: the Interition Solid server your Pod is on (crawlout.io, or a server Interition runs for you).

- **Sign-in.** Your agent's credential is exchanged with that server for a short-lived token.
- **Memory.** When Claude saves a memory, that entry is written to your Pod. Conversations, prompts and tool output are never sent.
- **Loading memory.** At session start, your saved entries are read from your Pod.
- **Team.** Channel posts, work records and documents are read and written only when you or Claude use those commands.
- **No other destinations.** No analytics, no telemetry, no third parties.

## Licence

Free to install and use with Interition's servers, under the Interition Plugin Licence in `LICENSE`. The open-source components it includes keep their own licences, listed in `THIRD-PARTY-NOTICES`.
