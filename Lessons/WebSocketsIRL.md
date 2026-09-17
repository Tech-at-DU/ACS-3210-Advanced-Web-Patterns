<!-- Run as a slideshow: reveal-md Lessons/WebSocketsIRL.md -w -->
# WebSockets in Real Life — Day 7

⭐️ **GOAL:** Leave able to choose WebSockets vs polling vs SSE for a real feature, name reconnect and firewall failure modes, and push a Make Chat / challenge MVP that uses Socket.IO for two-way traffic.

<!-- omit in toc -->
## ⏱ Agenda

- [[**5m**] Attendance &amp; Announcements](#5m-attendance--announcements)
- [[**15m**] ☀️ Warm Up](#15m-️-warm-up)
- [[**35m**] 📚 TT: Overview](#35m--tt-overview)
- [[**10m**] 🌴 Break](#10m--break)
- [[**20m**] 💻 Activity 1: Finish Make Chat Core Path](#20m--activity-1-finish-make-chat-core-path)
- [[**30m**] 💻 Activity 2: Challenge MVP or Whiteboard Inspiration](#30m--activity-2-challenge-mvp-or-whiteboard-inspiration)
- [[**5m**] Wrap Up](#5m-wrap-up)

<!-- > -->

<!-- omit in toc -->
## 🏆 Objectives

*By the end of this session, you'll be able to&hellip;*

1. **Compare** short polling, long polling, WebSockets, and SSE — and pick the fit for a one-way vs two-way feature.
2. **Explain** why corporate firewalls and dropped sockets need explicit reconnect logic (libraries help; they are not magic).
3. **Reuse** Day 6 wiring: Socket.IO on `http.Server`, right `emit` audience, handlers in a module.
4. **Ship** progress on Make Chat Tutorial and/or start [`Challenges/Websockets.md`](../Challenges/Websockets.md) with a clear done state.

<!-- > -->

## [**5m**] Attendance &amp; Announcements

Day 6 was the phone-call mental model and `http.Server` + Socket.IO. Today is **when that tool is the right call**, what breaks in the wild, and **lab time** on Make Chat + the WebSockets challenge. Challenge homework can finish after hours — tonight’s gate is a visible realtime MVP slice.

<!-- > -->

## [**15m**] ☀️ Warm Up

Skim (or revisit) [HTTP vs WebSockets: a performance comparison](https://blog.feathersjs.com/http-vs-websockets-a-performance-comparison-da2533f13a77) for **five minutes**. Then answer in chat or unmute:

1. Roughly where did the HTTP benchmark top out vs the WS one (order of magnitude is enough)?
2. In your own words: what about the connection model explains that gap?

If the article is slow to load, use this table instead and still answer #2:

| Style | Pattern | Feels “live” when… |
| --- | --- | --- |
| Short poll | Client asks on a timer | The timer fires — even if nothing changed |
| Long poll | Server holds until there is news | A message arrives, then client reconnects |
| WebSocket | Upgrade once, frames both ways | Either side pushes |
| SSE | Server → client stream over HTTP | Server pushes; client mostly listens |

**Warm-up prompt:** Name one product feature that is **one-way push** (SSE / poll might win) and one that is **two-way** (WS earns its keep).

> **📈 PROTIP:** “Live UI” is not automatically a socket problem. A receipt page that loads once is still HTTP.

<!-- > -->

## [**35m**] 📚 TT: Overview

**Next action:** Without sockets → WS tradeoffs → modern alternatives → Day 6 wiring recall → lab brief.  
**Done when:** You can recommend poll / SSE / WS for a ticket and name one reconnect failure mode.

### 1. Before WebSockets — The Old Tricks (~8m)

**Short polling:** timer in the browser, request every N seconds, update the DOM even when empty. Simple. Burns the server. Never truly in sync.

**Long polling:** request stays open until the server has something (or times out). Browser opens the next request immediately. Better latency when messages are rare; still one HTTP conversation per message burst; headers and auth tax every time.

```js
async function waitForDataFromServer() {
  const response = await fetch("/updates");
  const el = document.getElementById("latest-updates");

  if (response.status === 200) {
    el.textContent = "NEW: " + (await response.text());
    await waitForDataFromServer();
  } else {
    el.textContent = "ERROR: " + response.statusText;
    await new Promise((r) => setTimeout(r, 1000));
    await waitForDataFromServer();
  }
}
```

> **💬 ASK QUESTION:** Messages arrive about once an hour. Is long polling unreasonable?

<details>
<summary>Answer</summary>

**Often fine.** Rare messages + hold-until-data is exactly where long polling shines. High-frequency fanout is where it gets expensive.

</details>

### 2. Why Teams Still Pick WebSockets (~8m)

- Bidirectional, full-duplex after the upgrade (`101` then WS/WSS frames — Day 6)
- Server can push without waiting for the next client poll
- Binary or text frames; custom events (Socket.IO) without inventing a new HTTP route per ping
- Great fit: chat, presence, collab cursors, multiplayer, live boards

Costs you must say in review:

- Longer-lived sessions — overkill for a one-shot form
- Packet-inspecting firewalls (common on corporate networks) may kill WS; you need fallback or a clear error UX
- Reconnect is **your** problem (or your library’s). Dropped sockets do not auto-heal unless you implement it
- Auth and sticky sessions get real once you have more than one server

> **💬 ASK AUDIENCE:** A dashboard only needs server → browser stock ticks. Client never sends after subscribe. Is WebSocket the best default?

<details>
<summary>Answer</summary>

**Not always.** **SSE** (or even authenticated poll) is often enough for one-way push — simpler mental model, HTTP-friendly infrastructure. Use WS when the client must talk back on the same long-lived channel.

</details>

### 3. Modern Alternatives (Pick With Intent) (~7m)

| Need | Lean toward |
| --- | --- |
| Classic request/response | HTTP / REST |
| Server → client only | SSE, or a message queue + poll |
| Both directions, low latency | WebSockets (raw or Socket.IO) |
| Cross-service async work | Queues (e.g. SQS) — not a browser socket |

We skip inventing a custom protocol tonight. Socket.IO is the seatbelt: reconnect helpers, fallbacks, named events. Raw `WebSocket` is fine when you want the metal.

> **💬 QUICK CHECK:** Day 6 used `io.emit` for chat. Which alternative emit keeps the sender from getting their own message on the wire?

<details>
<summary>Answer</summary>

**`socket.broadcast.emit`** — everyone except the sender. Or append locally and broadcast. Day 6 stretch covered this.

</details>

### 4. Day 6 Wiring — 60-Second Recall (~5m)

- `http.createServer(app)` then `new Server(server)` — **`app.listen` will not mount Socket.IO**
- Named handlers, close over `socket`, guard payloads, `module.exports` a `registerChatHandlers(io)`
- Audiences: `socket.emit` / `socket.broadcast.emit` / `io.emit` (+ rooms later with `join` / `to`)

> **💬 YOUR TURN:** You open `index.html` via `file://` and the socket never connects. First fix?

<details>
<summary>Answer</summary>

Serve it from the **HTTP server** (`http://localhost:3000`). Same-origin Socket.IO script + handshake expect a real origin, not a file path.

</details>

> **‼️ CAUTION:** Mixing string payloads and `{ id, text }` objects without matching the client `textContent` line looks like “sockets are broken.” Match both sides.

### 5. Lab Brief — Imperfect and Honest (~2m)

If Make Chat is half done, finish the core path before inventing a second app. Challenge MVP can be chat *or* something else realtime — whiteboard inspiration is fine. We skip polishing reconnect tonight unless you are ahead.

<!-- > -->

## [**10m**] 🌴 Break

Leave `node` running if you want. Stretch. Come back ready to ship a visible slice.

<!-- > -->

## [**20m**] 💻 Activity 1: Finish Make Chat Core Path

> **✅ DONE WHEN:** Two browser tabs on `http://localhost:3000` share a message, handlers live in a module (or you are one clear step from extracting), and the terminal shows connect + at least one event.

1. Open your Day 6 Make Chat folder (or restart from [Socket.IO chat getting started](https://socket.io/get-started/chat) on Express 5 + CJS).
2. Confirm `http.createServer(app)` + `new Server(server)` + `server.listen(3000)`.
3. Confirm client uses `/socket.io/socket.io.js` and `const socket = io();` — not `file://`.
4. Get **two tabs** updating. Prefer `io.emit` or the broadcast-and-append pattern — pick one and stick to it.
5. If handlers are still inline, extract `registerChatHandlers(io)` into `sockets/chat.js` and `require` it from the entry file.

> **📈 TIP:** Unique `package.json` `"name"` — not `socket.io` / `express` — avoids install confusion.
> **FINISHED EARLY?** Add a connect/disconnect system line, or a `{user} is typing` stretch from the official homework list.

<!-- > -->

## [**30m**] 💻 Activity 2: Challenge MVP or Whiteboard Inspiration

> **✅ DONE:** You either (A) have a started challenge MVP with one realtime behavior beyond the tutorial echo, or (B) have a written 5-line plan + first commit toward [`Challenges/Websockets.md`](../Challenges/Websockets.md). Chat again is fine; a different realtime MVP is fine.

1. Skim [`Challenges/Websockets.md`](../Challenges/Websockets.md). Note the MVP bar for full credit.
2. Pick **one** direction:
   - Extend Make Chat (nicknames, rooms sketch, typing, presence)
   - Use inspiration from [socket.io whiteboard demo](https://socket.io/demos/whiteboard/) / [examples/whiteboard](https://github.com/socketio/socket.io/tree/master/examples/whiteboard) — draw events over the socket
3. Write the done state in one sentence at the top of your notes before coding.
4. Ship the smallest slice that proves two-way traffic for *your* feature.
5. Homework: finish the challenge after hours if the MVP is not complete tonight.


> **‼️ WATCH OUT:** Do not start a new framework. Same Express + Socket.IO stack as Day 6.
> **FINISHED EARLY?** Trade a 60s demo with a teammate: show the two-client proof.

<!-- > -->

## [**5m**] Wrap Up

> **📈 TIP:** One feature that works on two clients beats five half-wired events.

Session takeaways:

1. Poll when rare/simple; **SSE** for one-way push; **WS** when both sides talk.
2. Firewalls and drops → **reconnect UX** is part of the product.
3. Socket.IO still mounts on **`http.Server`**; emit audience still matters.
4. Challenge path: [`Challenges/Websockets.md`](../Challenges/Websockets.md) — finish after hours if needed.

Optional notes card:

```text
Shipped today:
Stuck on:
Next session first 15m:
```

<!-- > -->

## Additional Resources

1. **[Socket.IO — Get started (chat)](https://socket.io/get-started/chat)** — Express + `http.Server` + emit path.
2. **[Socket.IO — Server initialization](https://socket.io/docs/v4/server-initialization/)** — why `app.listen` fails for Socket.IO.
3. **[Socket.IO — Emitting events](https://socket.io/docs/v4/emitting-events/)** — audiences, acknowledgements, timeout.
4. **[MDN — WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)** — native WS / WSS.
5. **[javascript.info — Long polling](https://javascript.info/long-polling)** — hold-until-data pattern.
6. **[Ably — WebSockets vs SSE](https://ably.com/blog/websockets-vs-sse)** — one-way vs two-way choice.
7. **[Feathers — HTTP vs WebSockets performance](https://blog.feathersjs.com/http-vs-websockets-a-performance-comparison-da2533f13a77)** — warm-up article.
8. **[Challenge — Websockets](../Challenges/Websockets.md)** — course challenge brief.

## For Curriculum Authors

<details>
<summary>For Curriculum Authors</summary>

### In Class

| | |
| --- | --- |
| **Next action** | Agenda → Attendance → Warm Up article/table → TT tradeoffs → lab. |
| **Done when** | At least one two-tab realtime demo in the room, or a clear written MVP plan per person. |

- Warm-up does not need breakouts; chat answers are enough.
- Behind at ~0:35? Cut alternatives table detail; protect Activity time.
- Do not require the full challenge tonight — Activity 2 gate is a started MVP or a 5-line plan + first commit.

### Facilitator Notes

- Continuity with Day 6: same stack, same `http.Server` rule, same emit vocabulary.
- Prefer speakable TT; one failure story (firewall / no reconnect) beats a long disadvantages list.
- Keep ASK AUDIENCE pulses; they replace digressions.
- Whiteboard demo is optional inspiration — do not force a canvas app on everyone.

### Expert Follow-Ups

- Optional after-session: Socket.IO rooms; sticky sessions / Redis adapter (name only unless asked).
- Re-check Socket.IO current release notes day-of if pinning versions in a handout.

</details>
