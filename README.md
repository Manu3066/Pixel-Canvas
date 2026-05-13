# 🎨 Pixel Canvas

A real-time collaborative pixel art canvas where multiple users can draw together simultaneously — inspired by Reddit's r/place.

**[Live Demo →](pixel-canvas-production-ed55.up.railway.app)** <!-- replace with your Railway URL -->

![Pixel Canvas Demo](https://via.placeholder.com/800x400/0f0f11/7c6fff?text=Add+a+GIF+of+your+canvas+here)

---

## What it does

- 🖌️ **Draw pixels** on a shared 64×64 canvas in real time
- 👥 **Multiple users** see each other's changes instantly — no refresh needed
- 🏠 **Rooms** — create named canvases and share the room name with friends
- ⏱️ **Rate limiting** — one pixel per 200ms per user (just like r/place)
- 🔄 **Persistent state** — join mid-session and see the current canvas immediately
- ⚔️ **Conflict resolution** — simultaneous edits on the same pixel are resolved automatically

---

## Architecture

```
Browser (Tab 1)          Browser (Tab 2)
     │                        │
     │  WebSocket              │  WebSocket
     └──────────┬─────────────┘
                │
         Node.js Server
         (Express + ws)
                │
         In-memory state
         rooms[roomId][y][x]
         = { color, timestamp }
```

### Why WebSockets over HTTP polling?

HTTP polling would require each client to ask the server "anything new?" every few hundred milliseconds — wasteful and slow. WebSockets keep a persistent connection open, so the server can **push** updates to clients the instant they happen. This gives true real-time feel with minimal overhead.

### Conflict resolution — Last Write Wins

When two users paint the same pixel simultaneously, both events race to the server. Each pixel event carries a **client timestamp**. The server compares the incoming timestamp against the stored one:

```javascript
if (existing && existing.timestamp > incomingTimestamp) {
  // reject — existing pixel is newer
  ws.send({ type: 'pixel_reject', x, y, color: existing.color });
  return;
}
// accept — store and broadcast
room[y][x] = { color, timestamp };
broadcastToRoom(roomId, { type: 'pixel', x, y, color, timestamp });
```

The losing client receives a `pixel_reject` event and reverts its local pixel to the winner's color. This is a simplified **Last Write Wins (LWW)** strategy — the same approach used in distributed databases like Cassandra.

### Room isolation

Each room is an independent canvas stored in memory. WebSocket broadcast is scoped to clients in the same room — a pixel event in room `cats` never reaches a client in room `default`.

```javascript
function broadcastToRoom(roomId, msg, sender) {
  wss.clients.forEach((client) => {
    if (client.roomId === roomId && client !== sender) {
      client.send(JSON.stringify(msg));
    }
  });
}
```

### Rate limiting

Each WebSocket connection tracks its last paint timestamp server-side. Incoming pixel events that arrive faster than 200ms are silently dropped — preventing spam and canvas flooding.

```javascript
const now = Date.now();
if (now - lastPaint.get(ws) < 200) return; // too fast, ignore
lastPaint.set(ws, now);
```

---

## Tech stack

| Layer | Technology |
|---|---|
| Runtime | Node.js |
| HTTP server | Express |
| Real-time | WebSockets (`ws` library) |
| Frontend | Vanilla HTML5 Canvas + JS |
| Deployment | Railway |

---

## Running locally

```bash
git clone https://github.com/YOURNAME/pixel-canvas
cd pixel-canvas
npm install
node server.js
```

Open `http://localhost:3000` in two browser tabs and draw in one — changes appear in the other instantly.

To test rooms: type different room names in each tab and click **Join**.

---

## Design decisions & trade-offs

**Why not Redis pub/sub?**
Redis pub/sub would allow multiple Node.js server instances to stay in sync — essential for horizontal scaling. For this project, in-memory state is sufficient since it runs as a single instance. Redis integration is the natural next step before scaling.

**Why not CRDTs?**
Conflict-free Replicated Data Types (CRDTs) would eliminate conflicts entirely by design, but add significant complexity. For a pixel canvas where the most recent human intent should win, Last Write Wins is simpler and appropriate.

**Why vanilla Canvas over a framework?**
The pixel grid is a performance-sensitive render loop. Direct Canvas 2D API gives full control over what redraws — only the affected cell is redrawn on each pixel event, not the entire canvas.

---

## What I'd add next

- [ ] Redis pub/sub for horizontal scaling across multiple server instances
- [ ] Persistent storage (PostgreSQL) so the canvas survives server restarts
- [ ] Shareable room links
- [ ] User cursors showing where others are hovering
- [ ] Canvas export as PNG

---

## Author

Built by **[Manasa Pillai](https://github.com/Manu3066)** as a distributed systems learning project.
