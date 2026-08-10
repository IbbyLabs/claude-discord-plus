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

**`describe_server`.** Every channel and category with its id and type, every
role with its id, position and colour, the custom emoji `react` can use, the
member count and the guild name — in one call:

```
IbbyLabs  (guild id: 1339…, 412 members)
channels (23):
#general  (id: 1339…, text, in: Community)
#bugs  (id: 1382…, forum, in: Support)
roles (6):
@Maintainer  (id: 1341…, pos: 4, #5865f2)
emoji: <:xrdb:1355…> <a:shipit:1361…>
```

The bridge could read every message in a channel and could not say what channels
existed, so posting anywhere nobody had written from meant asking a human for the
id. Pass any allowlisted channel in the guild. The answer is cached for a few
minutes — channel and role lists change rarely, and this is meant to be cheap
enough to call before guessing.

**Embed content.** Bots and webhooks put the whole message in an embed and leave
`content` empty, so upstream renders the most informative messages as blank
lines. Title, description, fields and footer are now included:

```
[2026-07-30T04:00:00.000Z] Community Bot: [embed: [BUG-172] Incorrect logo language is selected · Severity: Medium · Status: New]  (id: 111)
```

That line used to be empty. Identifiers living in an embed or a thread title
were unreadable, which meant reaching into the bot's database to recover them.

The same flattening runs on the way in, so a message that arrives from a channel
carries its embed rather than nothing:

```
[embed: Container Start [IbbyLabs] · Container "xrdb" start (ghcr.io/ibbylabs/xrdb:latest) · Environment: IbbyLabs]
```

A #github or #updates channel delivers without needing a mention, and everything
posted there is embed-only, so an empty `content` was the whole message. The
embed goes in the content, where the message belongs — the attachment listing
stays in the envelope meta, because an in-content annotation is forgeable by
anyone who types it.

**`search_messages`.** Wraps `GET /guilds/{guild.id}/messages/search`, available
to bots since August 2025. Filters on content, author, channel, `has`, pinned
and sort order.

Search spans a whole guild, so the allowlist goes into the query as a
multi-valued `channel_id` filter — Discord takes up to 500, and a forum's id
covers its posts. Checking each result instead would find nothing in a tracker:
a hit inside a forum post carries the post's own channel id, while the allowlist
is keyed on the forum. Filtering in the query also stops channels the caller
cannot read from spending the result budget.

Discord caps one request at 25 results. `limit` goes to 200 and pages
underneath, `offset` continues from an earlier call, and the reply names the
offset to pass next when more matched than came back.

**Thread, reply and attachment context.** `fetch_messages` now reports the thread
name, the referenced message id, and attachment filenames with content types
instead of a bare `+2att`.

**`delete_message` and `pin_message`.** Housekeeping the bot could not do.

**`list_forum_threads`.** Lists a forum's posts with their id, author, status
tags, archived state, message count, and when each was opened and last active,
covering active and archived threads. Upstream can only see a post it was sent a
message from, so everything nobody wrote in was invisible — which is most of a
tracker. This is what makes triage possible from the outside rather than waiting
to be told.

```
Logo language is wrong  (id: 1532661861219045387, by: alice, tags: Confirmed, archived, 14 msgs, opened: 2026-07-02T09:14Z, last: 2026-07-28T16:02Z)
```

The timestamps are what a staleness sweep runs on: deciding a post has gone
quiet used to mean opening every one of them. An author who is not already in
the cache is reported by id rather than costing a lookup per post.

**`create_forum_post`.** Opens a post: a title, an opening message, and
optionally the tags it starts with. Nothing here could open one, so a bridge
documented as filing bug reports actually asked a human to create the post.
Returns the new thread's id and a jump URL.

**`close_thread`.** Archives, locks, unarchives, unlocks, and sets the `PINNED`
channel flag so a FAQ post sticks to the top of its forum. A tracker where
nothing ever closes only grows, and `set_thread_tags` deliberately reopens
archived threads to tag them, so something has to put them back. Asking for a
state a thread already holds changes nothing and is not an error, so a sweep can
close everything it triaged without checking first.

**`forward_message`.** Forwards a message to another channel using
`message_reference` type 1, which carries `message_snapshots` — the text, embeds
and attachments as Discord renders them. Escalating by retyping the report loses
exactly the screenshot the report was about. A forward carries no text of its
own, so an optional note is posted first as its own message.

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

**Error log.** Failures are appended to `errors.log` next to `access.json`, in
addition to stderr. The bridge's stderr belongs to whatever launched it, so a
throw while handling an inbound message leaves no copy anyone can read, and a
message that silently never arrives is indistinguishable from one nobody sent.
The file is capped at 256 KB and truncated when it exceeds that.

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
the ack reaction would come straight back at it.

Events follow the channel's delivery mode, the same as its messages. A channel
that delivers everything relays every event. A mention-only channel relays a
reaction on one of the session's own messages, which is how an answer gets
acknowledged without typing, and nothing else — a reaction on a stranger's
message there is a wake-up carrying nothing to act on. A channel with no
explicit `groups` entry relays nothing. Member and voice events come from any
guild holding an entry.

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

**Voice notes are transcribed.** A voice note arrives as an ogg attachment the
session cannot open, so its content is lost and the only possible reply is
asking the sender to type it out. The first audio attachment on an inbound
message is now transcribed and the text appended to the content:

```
[voice note: "the render came out blank again"]
```

The transcript is also in the envelope meta as `transcript`, with
`transcript_language` when the transcriber reports one. A message with no text
becomes the transcript alone.

An attachment counts as audio when its content type starts with `audio/`, or
when Discord marks the message with the voice-message flag. Only the first one
is transcribed, so ten notes in a message do not hold delivery for minutes.

A note too long to transcribe in one run is split with `ffmpeg` into
`DISCORD_TRANSCRIBE_CHUNK_SECONDS` pieces (20 by default), each transcribed
inside its own timeout and joined in order, so length is not a ceiling. The
default is well under the timeout because transcription runs at roughly real
time on a busy machine, and a piece that finishes near the limit is one spike
away from being dropped. A piece that fails leaves a `[transcription failed]`
gap so the rest of the note still arrives, and a note
where every piece fails becomes `[voice note: could not be transcribed:
<reason>]` with the reason carried through. Splitting needs `ffprobe`/`ffmpeg`
on the path (overridable with `DISCORD_FFPROBE`/`DISCORD_FFMPEG`); without them a
short note still transcribes in one pass.

Transcription runs an external command, `DISCORD_TRANSCRIBER`, defaulting to
`~/.claude/bin/transcribe-audio.py`. It is given the downloaded file as its only
argument and is expected to print the transcript on stdout, optionally prefixed
with `[en]`. The file is removed afterwards. No such command means no
annotation, so a bridge without a transcriber behaves as before.

Failure is never silent. A non-zero exit, or a run past
`DISCORD_TRANSCRIBE_TIMEOUT_MS` (60s by default, after which the child is
killed), appends `[voice note: could not be transcribed]` and puts the reason in
`transcript_error` and on stderr. The message is delivered either way.

**"Report as bug" and "Report as feature request", on right-click.** Two message
context-menu commands, registered on connect. Picking one on a support question
or a suggestion sitting in #general hands that message to the session — its text,
its author, its jump URL and what it has attached — instead of asking whoever
noticed to go and describe it in the tracker themselves. `report_kind` on the
event says which of the two was picked, so the session knows which forum it
belongs in and who to credit.

Discord discards an interaction nobody answers within three seconds and a
session is minutes away, so the interaction is deferred immediately and
acknowledged privately to whoever clicked. The real outcome then arrives as an
ordinary message in the channel, not an interaction followup: the token dies
after fifteen minutes and a session routinely runs longer than that.

The command needs the `applications.commands` scope on the bot's invite. Without
it registration fails and is logged, and the entry never appears. Who may use it
is Discord's to answer — Server Settings → Integrations restricts a command by
role or channel.

**Mass mentions, on request only.** The client parses user mentions and nothing
else, so `@everyone` typed at the bot — in a DM it mirrors, in a channel it
relays — notifies nobody. That stays true regardless of what any message says.

`reply` takes a `mentions` array for the case the operator actually wants:
`"everyone"`, `"here"`, or `"role:<role id>"`.

```json
{ "chat_id": "…", "text": "v3.47.0 is out.", "mentions": ["everyone"] }
```

Omitted, nothing beyond user mentions is parsed. Present, the message is
permitted exactly what was named — `everyone` and `here` both map to the
`everyone` parse, and a role is passed by id in the `roles` list. `parse` never
carries `roles`, which would permit every role named anywhere in the text. The
opt-in is the parameter; message content cannot reach it.

**Polls.** `reply` takes a `poll`, sending a native Discord poll under the text:

```json
{
  "chat_id": "…",
  "text": "Which one ships first?",
  "poll": {
    "question": "Which one ships first?",
    "answers": ["Folder writing", "Community themes"],
    "duration": 48,
    "allow_multiselect": false
  }
}
```

Two to ten answers, `duration` in hours (24 by default, 768 at most), and
`allow_multiselect` for voters picking several. Counts, lengths and duration are
checked here, so a poll that Discord would reject with a 400 comes back saying
which limit it broke. Apps cannot vote in their own polls, so this collects other
people's answers and holds no opinion of its own.

It rides on `reply` rather than being its own tool: a poll is a message with a
question attached, and every tool's schema is paid for on every turn.

**Message links.** People paste `https://discord.com/channels/…` constantly, and
the bridge could do nothing with one. `fetch_messages` now takes a link anywhere
it takes a channel:

```
fetch_messages(channel: "https://discord.com/channels/1339…/1382…/1401…")
```

A link naming a message returns that message with the conversation either side of
it, the linked one marked `→`. A link naming only a channel reads it as an id.

The link resolves to ids and then goes through the same allowlist check
everything else does, so pasting one is a shorter way to write a channel id and
not a way around what reading is allowed to reach.

**Typing while it thinks.** A turn takes minutes and Discord holds a typing
indicator for about ten seconds, so the channel used to sit silent from the
moment a message arrived until the reply landed. The indicator now starts when a
message is accepted and is re-sent until the reply goes out.

There is no tool for it and it costs no tokens. One timer per channel and never a
second — a further message refreshes the existing deadline rather than scheduling
again. Every timer carries an absolute deadline that stops it, a failing send
clears it, and shutdown clears them all, so the worst case is a channel that
types for ten minutes and then stops itself.

**Reminders.** Discord has no reminder feature, so `remind` is one: a due time, a
channel and a note, posted when it comes due.

```json
{ "action": "create", "chat_id": "…", "when": "friday 9am", "note": "chase the release PR" }
```

`when` reads a relative form (`in 2 hours`, `90m`, `1h 30m`), a day and time
(`tomorrow 9am`, `friday 17:00`), or an absolute timestamp, in the host's local
zone. Anything it cannot read is rejected with the accepted forms rather than
guessed at, because a misread reminder fires at the wrong time and says nothing.
`action: "list"` reports what is scheduled with ids, `action: "cancel"` takes one.

The store is `reminders.json` next to `access.json`, so reminders survive a
restart, and a due time that passed while the bridge was down fires when it comes
back. A corrupt store is moved aside on read and the process starts with none,
rather than failing to boot. The tick runs once a minute; the target channel is
checked against the allowlist both when the reminder is set and again when it
fires, since the allowlist may have changed in between.

**Bot messages use the bot's name.** Text a Discord user reads — the pairing
confirmation, the answers to "Report as bug" — names the bot by its own display
name rather than naming Claude, since this runs under whatever name the
application was given. The pairing instruction is the exception and still says
Claude Code, because it names the program the operator has to open.

### Reply context

An inbound reply carries the message it answers — author and an excerpt of the
content — as `[replying to <who>: "..."]`, plus `reply_to_message_id` and
`reply_to_user` in the meta. Discord sends only a reference id, so without this
the session sees the reply and not what it replies to.

An outbound reply notifies the person it answers. A reply split across chunks
notifies once, on the chunk carrying the reference.

### A withdrawn message withdraws its attachment

`download_attachment` records what it pulled and from which message. When that
message is deleted, the files go with it. Someone pasting a config and thinking
better of it is the case this exists for — without it the sender's deletion
removes the message from Discord and leaves the copy on disk.

### Who sent a history line

`fetch_messages` and `search_messages` name each sender as `username (id)`. A
live message carries `user_id` in its envelope, so without the id in history
anything read back — including after a restart — could only be identified by a
display name, which its owner chooses. The id is evidence Discord supplied, not
proof against a doctored transcript: good enough to recognise someone, not
enough to act on something consequential without live confirmation.

### Where a message keeps its text

`content` is only one of the places a Discord message says something, and a
message that says nothing there is not an empty message. Each of the following
is read on the way in, in history through `fetch_messages` and `search_messages`,
in the excerpts on reaction, edit and delete events, and in the replied-to
message:

**Mentions read as names.** `@alice can you check this` arrives on the wire as
`<@1147968681981247498> can you check this`, which names nobody. Content now
carries the readable form, with user, role and channel mentions resolved:

```
@alice can you check this in #bugs
```

The ids are still what a tool call and a mention back need, so they travel in
the meta as `mentioned_users`, `mentioned_roles` and `mentioned_channels`, each
a `name=id` list.

**Forwarded messages.** A forward carries its text in `message_snapshots` and
leaves `content` empty, so forwarding something to the bot used to deliver a
blank message. The snapshot's text, embeds, components, stickers and attachment
names are rendered, and the origin travels as `forwarded_from_message_id` and
`forwarded_from_chat_id`:

```
[forwarded: renderer fell over again on the 4k poster · attachments: log.txt]
```

**Components-v2 messages.** A message sent with the components-v2 flag has no
`content` and no embeds at all; its text lives in the component tree. The tree
is walked and the text displays are collected in order, with button and select
labels after them:

```
[components: **Transcribed voice message from Ibby** · The message where I replied shouldn't show as empty. · buttons: Play / Dismiss]
```

**Polls and stickers.** A poll's question and options with their vote counts,
and the name of a sticker sent on its own:

```
[poll: Ship it today? — yes (4) / tomorrow (1)]
[sticker: thumbs up]
```

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
