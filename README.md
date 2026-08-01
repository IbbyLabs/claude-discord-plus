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

**`list_forum_threads`.** Lists a forum's posts with their id, status tags,
archived state and message count, covering active and archived threads. Upstream
can only see a post it was sent a message from, so everything nobody wrote in was
invisible — which is most of a tracker. This is what makes triage possible from
the outside rather than waiting to be told.

**`fetch_messages` pages.** Discord caps a single request at 100 and rejects
anything higher, so a longer thread is read by walking back from the oldest id
returned. `limit` goes to 1000, and `before` continues from an earlier call.

**Thread name in the envelope.** An inbound message now carries `thread_name` and
`parent_id`. Trackers keep the report id in the post title (`BUG-171`,
`FR-126`) and nowhere else in the payload, so without it the id that closes a
report cannot be read from the message at all.

**Gateway state file.** A live socket is not a live gateway: the connection can
stay open while nothing is delivered, which looks exactly like a quiet channel.
The server writes `gateway.state` next to `access.json` every 20 seconds with the
websocket status, ping and the time it last heard anything from Discord, so a
watchdog can check whether it is actually receiving rather than merely connected.

**Bot and webhook messages are delivered.** Upstream drops every message whose
author is a bot, which hides the channels that report what shipped — releases,
issues and commits all arrive by webhook. Only the bot's own messages are
skipped, so it cannot answer itself.

**Forum tags.** `list_forum_tags` reads a forum's tags with their ids;
`set_thread_tags` sets a thread's tags by name or id. On a tracker forum the tags
are the statuses, so this is a control surface rather than decoration — the bot
watching the forum turns a tag change into a status change.

`ensure_forum_tags` adds tags a forum does not have yet. Discord replaces the
whole tag list on write, so it sends the existing tags back with the new ones —
a forum's tags carry threads, and dropping one strips every thread using it.
Names match case-insensitively, so running it twice is a no-op.

Discord rejects edits to an archived thread, which is most of a tracker's
history, so `set_thread_tags` reopens one and applies the tags, saying in the
reply that it did. It stays open: a bot reacting to the tag edits the thread, and
re-archiving immediately beats it to the post. A forum with no tags defined says
so rather than reporting the tag you asked for as unknown.

Forum channels are not text-based, so `fetch_messages` cannot read a forum
itself; read its threads instead. The tag tools resolve the forum from either the
channel or a thread inside it, and check the same allowlist.

**Ambient events.** Upstream connects with the message intents alone, so a
thumbs-up, a correction typed into an existing message, a deletion and everyone
arriving or leaving are all invisible. These now arrive as one line each on the
same notification channel as a message, with an `event` in the meta and a tag in
the text:

```
[reaction+] alice reacted 👍 to 1399 (me: "shipped in v3.14.2")
[edit] alice edited 1399: "port 8080" → "port 8081"
[delete] alice's message 1399 deleted: "wrong channel, sorry"
[member+] alice joined
[member~] alice roles +Beta -Trial; nickname (none) → "Al"
[voice~] alice moved General → Standup
```

Only content edits are relayed, because Discord fires the same event for embed
resolution, link unfurls and pins. Voice reports joins, leaves and moves only,
not mute, deafen or stream toggles. Reactions the bot adds itself are skipped, or
the ack reaction would come straight back at it. Events relay from channels with
an explicit `groups` entry, and member and voice events from any guild holding
one.

**`list_members`.** Members of a guild with their status, roles and nickname,
filterable by `status` or `role`. Presence is deliberately pull-only: the
`presenceUpdate` event fires on every status flicker of every member, which would
bury the conversation, so the intent is enabled to populate the cache and the
roster is read on demand instead.

**`defaultPolicy`.** A channel with no `groups` entry used to drop everything,
which meant the bot could be @mentioned anywhere it had been invited and answer
nowhere. `defaultPolicy` is the policy those channels get, and defaults to
`{"requireMention": true}`: a mention reaches the session from any channel the
bot can see, while unlisted channels still do not deliver everything said in
them. An explicit `groups` entry always wins. Set it to `null` for the older
behaviour, where a channel is silent until it is listed.

Reading follows delivery: the tools reach a channel the inbound gate would
deliver from, so with a `defaultPolicy` set, `fetch_messages` and
`search_messages` cover those channels too. `null` closes both again.

**Open DMs, with a trace.** `dmPolicy` takes a fourth value, `guild`, which
accepts a DM from anyone sharing a guild with the bot and drops the rest. A
stranger's DM is otherwise dropped in silence, so a user who sends a config or an
ID to the bot gets nothing back and nobody learns they tried.

`dmMirrorChannelId` copies every accepted DM to a channel, since a DM is
otherwise visible to nobody but the bot:

```
📥 DM from alice (184695080709324800): the render came out blank again
```

A failed mirror is logged and the message still goes through. Inbound DMs carry
`is_dm` in the envelope so the session can hold a private message to a different
standard than something said in front of a channel.

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
