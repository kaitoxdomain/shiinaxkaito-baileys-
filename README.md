# ShiinaxKaito Baileys

High-level builders for WhatsApp **interactive messages**, **carousels**, and
**flow-style multi-step interactions** on top of [Baileys](https://github.com/WhiskeySockets/Baileys)
-- plus a small set of helpers for **staying in sync with WhatsApp protocol
changes** and **reducing avoidable ban risk** (rate-limited sending, health
monitoring, presence pacing).

> Formerly published as `baileys-interactive-toolkit`. Same code, new name
> and a few new modules -- see "What's new" below if you're upgrading from
> that version.

Built and type-checked against `baileys@7.0.0-rc14` (Aug 2026). Baileys ships
breaking changes across major versions, so pin your version and re-check
the type-checks in this repo after upgrading.

## Why a wrapper, not a fork

Baileys already maintains the hard, constantly-shifting part of this problem:
the binary-node / Noise-protocol layer that talks to WhatsApp's servers.
That layer changes whenever WhatsApp tweaks its wire protocol, and
WhiskeySockets tracks those changes upstream. Forking that layer means
manually chasing every future protocol fix yourself; depending on `baileys`
as a peer dependency means you inherit those fixes via a normal `npm update`.

This toolkit only adds the layer Baileys doesn't cover: ergonomic builders
for `interactiveMessage` / `carouselMessage` content, and a `relayMessage`
wrapper that attaches the additional protocol node these message types need
to actually render (see "The interactive-message gap" below).

## What's new in ShiinaxKaito Baileys

Everything from `baileys-interactive-toolkit` 0.1.0 works unchanged --
`InteractiveClient`, `relayInteractiveMessage`, and the builders all keep
the same signatures. This release adds two opt-in areas, both covered in
their own sections below:

- **`src/compat/`** — protocol-drift awareness: a local, no-network check
  that flags when your installed `baileys` version hasn't been verified
  against `relay.ts`, plus a helper for checking WhatsApp Web version drift
  on demand. See ["Staying in sync with WhatsApp's protocol"](#staying-in-sync-with-whatsapps-protocol).
- **`src/safety/`** — ban-risk-reduction helpers: a rate-limited send queue
  with an optional warm-up ramp, a health monitor that classifies
  disconnect codes and flags rising risk, and a presence-pacing helper.
  See ["Reducing ban risk"](#reducing-ban-risk).

Both are additive and off by default — `new InteractiveClient(sock)` behaves
exactly as before. Nothing here is a fork of Baileys itself or a change to
how it talks to WhatsApp's servers; see "Why a wrapper, not a fork" above.

## Install

```bash
npm install baileys shiinaxkaito-baileys
```

`baileys` is a peer dependency — bring whatever version your project
already uses. If your project depends on the scoped package name
(`@whiskeysockets/baileys`) instead of the unscoped `baileys` name, both
currently resolve to the same underlying project; just adjust the import
in the examples below to match whichever one you have installed.

### Note: this package prints a banner on import

`import`-ing anything from `shiinaxkaito-baileys` prints a small ASCII
banner (name + support/store contact info) to the console once, the first
time the module loads in a process — see `src/banner.ts`. This is a
deliberate, one-time side effect for attribution/support purposes, not a
bug. It runs before any of your own connection logic and doesn't touch
the socket, so it's safe to ignore if you don't want it, but there's
currently no flag to disable it.

## Quick start

```ts
import makeWASocket from 'baileys'
import { InteractiveClient } from 'shiinaxkaito-baileys'

const sock = makeWASocket({ /* ...your existing config... */ })
const interactive = new InteractiveClient(sock)

// wait for connection.update -> connection === 'open' before sending
await interactive.sendInteractive(jid, {
  body: 'Pilih salah satu:',
  buttons: [
    { type: 'quick_reply', displayText: 'Ya', id: 'yes' },
    { type: 'cta_url', displayText: 'Buka Website', url: 'https://example.com' }
  ]
})
```

See `examples/basic-usage.ts` for a full runnable example covering all
three message types wired into a real message handler.

## What's implemented

### 1. Interactive messages (`sendInteractive`)
Header (title/subtitle, optional image or video), body, footer, and buttons:
`quick_reply`, `cta_url`, `cta_call`, `cta_copy`, and `single_select` (list
menu with sections/rows). This is the direct replacement for the old
`buttonsMessage`/`templateMessage`, which WhatsApp stopped rendering for
non-Business-API senders.

### 2. Carousels (`sendCarousel`)
A swipeable set of cards, each with its own image/video, title, body, and
buttons. Cards are prepared concurrently since each one may involve a media
upload.

### 3. "Native flow" (`startFlow` / pseudo-flow) — **read the caveat**
**This is the one place where full WhatsApp Business API parity genuinely
isn't achievable outside the official platform.** A real WhatsApp Flow is a
multi-screen form hosted by Meta and exchanged over an encrypted channel
tied to a verified Business Account — there's no way for an unofficial
client to reference a Flow that doesn't exist on Meta's servers.

What this toolkit provides instead is a **pseudo-flow**: a sequence of
ordinary interactive messages chained together by your own handler logic,
so a multi-step interaction feels flow-like even though it's really just
several messages in a row. It's a UX pattern built from the interactive
message builder above, not a protocol implementation. Full detail and
reasoning is in `src/builders/flow.ts` — read it before you build on it.

## The interactive-message gap this toolkit patches

Baileys' protobuf types (`WAProto`) already define `interactiveMessage`,
`nativeFlowMessage`, and `carouselMessage`. But as of the version this was
checked against, `sock.sendMessage()`'s internal handling only adds special
protocol nodes for delete/edit/pin/poll/event content — nothing for
interactive messages. Send one through plain `sendMessage()` and it can
arrive with nothing visible on the recipient's device.

The fix (used by several community Baileys add-ons, not something
WhiskeySockets documents) is to build the message with
`generateWAMessageFromContent` and relay it with `sock.relayMessage()`
directly, attaching a `{ tag: 'bot', attrs: { biz_bot: '1' } }` node for 1:1
chats. That's what `relayInteractiveMessage()` in `src/relay.ts` does. This
was reverse-engineered from observed client behavior, not from any
published spec — if buttons or carousels stop rendering after a future
Baileys or WhatsApp update, `src/relay.ts` is the first place to check, and
the [WhiskeySockets Discord](https://whiskey.so/discord) / [baileys.wiki](https://baileys.wiki)
are the best places to look for what changed.

## Staying in sync with WhatsApp's protocol

WhatsApp's wire protocol and client behavior shift without notice, and this
toolkit's riskiest piece is `relay.ts` — a reverse-engineered workaround,
not a documented API (see "The interactive-message gap" above). `src/compat/version.ts`
gives you two checks instead of finding out a change happened only when a
user reports broken buttons:

```ts
import { warnIfBaileysVersionDrifted, checkWaWebVersionDrift } from 'shiinaxkaito-baileys'
import { fetchLatestWaWebVersion } from 'baileys'

// Local, no network call. Safe to run on every startup.
warnIfBaileysVersionDrifted()

// Network call -- run this from a periodic job or admin command, not on
// every connect. It tells you whether the WA Web version your socket is
// pinned to has drifted from what WhatsApp currently serves; it does not
// auto-apply anything.
const drift = await checkWaWebVersionDrift(fetchLatestWaWebVersion, myPinnedVersion)
if (!drift.pinnedIsCurrent) {
  console.warn(`pinned ${drift.pinned}, WhatsApp now serves ${drift.latest}`)
}
```

Neither check patches anything automatically — Baileys' own configuration
docs specifically advise against jumping to "latest" on every connect,
since your protobufs and `relay.ts` haven't been re-verified against it.
Treat a drift warning as "go check `relay.ts` and the WhiskeySockets
Discord / baileys.wiki before your next deploy," not as something to
auto-resolve unattended.

## Reducing ban risk

**Read this first:** nothing built on an unofficial client — this toolkit
included — can eliminate ban/restriction risk. Baileys itself says as much,
and WhatsApp's abuse detection is heuristic and undocumented, so treat
everything below as *reducing the odds of an avoidable restriction*, not a
guarantee. The single biggest avoidable cause reported across the Baileys
community is unpaced, bulk-looking sends to low-engagement contacts —
`src/safety/` targets exactly that.

### `MessageQueue` — pacing, rate caps, warm-up
A single-lane send queue with randomized delay between sends, an optional
per-minute cap, and an optional warm-up ramp for numbers that are new or
have been dormant (gradually raising the daily limit over a few days
instead of allowing full volume immediately):

```ts
import { MessageQueue, InteractiveClient } from 'shiinaxkaito-baileys'

const queue = new MessageQueue({
  minDelayMs: 1500,
  maxDelayMs: 4000,
  maxPerMinute: 20,
  warmUp: { startPerDay: 20, fullPerDay: 200, rampDays: 7 }
})

const interactive = new InteractiveClient(sock, { queue })
// sendInteractive/sendCarousel now route through the queue automatically
```

### `HealthMonitor` — turning disconnects into a risk signal
Classifies raw `DisconnectReason` status codes from your `connection.update`
handler (`forbidden`, `loggedOut`, `connectionReplaced`, etc.) and escalates
a risk level as they accumulate, so you can react — e.g. auto-`pause()` the
queue on a `forbidden` (403), the strongest single restriction signal —
before things get worse instead of after:

```ts
import { HealthMonitor } from 'shiinaxkaito-baileys'

const health = new HealthMonitor({
  onRiskChange: (level) => {
    if (level === 'high') queue.pause()
  }
})

sock.ev.on('connection.update', ({ connection, lastDisconnect }) => {
  if (connection === 'close') {
    const statusCode = (lastDisconnect?.error as { output?: { statusCode?: number } } | undefined)?.output?.statusCode
    health.recordDisconnect(statusCode)
  }
})
```

### `sendWithPresence` — natural typing pacing
Sends a `composing` presence, waits roughly as long as a person would take
to type the outgoing text, sends `paused`, then delivers the message. This
is ordinary `sock.sendPresenceUpdate` usage, not a protocol trick — it just
avoids the instant, always-identical-latency reply pattern that reads as
robotic:

```ts
import { sendWithPresence } from 'shiinaxkaito-baileys'

await sendWithPresence(sock, jid, body, () =>
  interactive.sendInteractive(jid, { body, buttons })
)
```

See `examples/production-setup.ts` for all three wired together with
`warnIfBaileysVersionDrifted()` on startup.

### What this toolkit deliberately does not do
No fingerprint/device-identity spoofing, no faking multiple real clients,
no content-mutation designed to slip past spam filters, and no attempt to
make automated traffic indistinguishable from a human specifically in
order to defeat WhatsApp's detection. Those cross from "pace your own
legitimate traffic sensibly" into actively working around WhatsApp's abuse
systems, which is a different thing this project won't help with. Beyond
the modules above: use a dedicated number rather than a personal one,
prioritize two-way conversations over one-way blasts (WhatsApp is reported
to track reply ratio), and get real opt-in before messaging someone first.

## Extending this

Folder layout:

```
src/
  builders/
    interactive.ts   — interactiveMessage + button builders
    carousel.ts       — carouselMessage builder
    flow.ts           — pseudo-flow session helper
  compat/
    version.ts        — baileys/WA Web version-drift checks
  safety/
    queue.ts           — rate-limited send queue + warm-up ramp
    health.ts           — disconnect classification + risk monitor
    presence.ts         — typing-pacing helper
  relay.ts             — the biz_bot relay wrapper
  client.ts            — InteractiveClient facade (queue-aware)
  types.ts             — ergonomic option types
  banner.ts            — console banner printed once on import
```

To add a new message type or button type: add a builder function in
`src/builders/`, extend the option types in `src/types.ts`, export it from
`src/index.ts`, and route sending through `relayInteractiveMessage()` so it
gets the same protocol-node handling as everything else.

## License

MIT
