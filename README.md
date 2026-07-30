# claude-discord-plus

Discord channel for Claude Code, extended so history is actually readable.

A derivative of the official [`discord`](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/discord)
plugin. Access control, pairing and the permission relay are upstream's and
unchanged — this adds the reading and housekeeping tools that were missing.

## Install

```
/plugin marketplace add IbbyLabs/claude-discord-plus
/plugin install discord-plus@claude-discord-plus
```

Set up access exactly as upstream: `/discord:access`. See [ACCESS.md](plugins/discord-plus/ACCESS.md).

## What it adds

**Embed content.** Bots and webhooks put the whole message in an embed and leave
`content` empty, so upstream renders the most informative messages as blank
lines. Title, description, fields and footer are now included:

```
[2026-07-30T04:00:00.000Z] Community Bot: [embed: [BUG-172] Incorrect logo language is selected · Severity: Medium · Status: New]  (id: 111)
```

That line used to be empty. Identifiers living in an embed or a thread title
were unreadable, which meant reaching into the bot's database to recover them.

**`search_messages`.** Wraps `GET /guilds/{guild.id}/messages/search`, available
to bots since August 2025. Filters on content, author, channel, `has`, pinned
and sort order.

Results are re-checked against the allowlist per hit. Search spans a whole
guild, so without that it would surface channels `fetch_messages` is not
permitted to read.

**Thread, reply and attachment context.** `fetch_messages` now reports the thread
name, the referenced message id, and attachment filenames with content types
instead of a bare `+2att`.

**`delete_message` and `pin_message`.** Housekeeping the bot could not do.

**Forum tags.** `list_forum_tags` reads a forum's tags with their ids;
`set_thread_tags` sets a thread's tags by name or id. On a tracker forum the tags
are the statuses, so this is a control surface rather than decoration — the bot
watching the forum turns a tag change into a status change.

Forum channels are not text-based, so `fetch_messages` cannot read a forum
itself; read its threads instead. The tag tools resolve the forum from either the
channel or a thread inside it, and check the same allowlist.

## Relationship to upstream

Forked from `cf99fc252a44e3f36763abe1db8744757f1b0297`, plugin version 0.0.4. Modifications are listed in
[NOTICE](NOTICE) as Apache-2.0 requires.

Upstream does not accept code from non-members
([their scope guard](https://github.com/anthropics/claude-plugins-official/blob/main/.github/workflows/close-external-prs.yml)
auto-closes such PRs), so the changes are offered there as
[issue #4678](https://github.com/anthropics/claude-plugins-official/issues/4678)
and shipped here in the meantime. If they land upstream, this fork should be
retired rather than maintained in parallel.

A weekly workflow watches upstream `server.ts` and opens an issue when it moves,
because the risk of a fork like this is silently missing a fix to the access
control it inherits.

## Licence

Apache-2.0, as upstream. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
