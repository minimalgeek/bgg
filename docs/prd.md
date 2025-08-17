# Type-Safe Multiplayer Board Game Engine

## 1. Vision

Provide a **type-safe, deterministic, multiplayer board game engine** in TypeScript that:

- Shares code and types between backend and frontend.
- Supports **complex board game flows**: turns, phases, simultaneous actions, hidden information.
- Uses **functional programming**: no classes, immutable state via **ImmerJS**.
- Supports **optimistic updates** while server remains authoritative.
- Fully **replayable** using RNG seeds + action logs.
- Designed for **Next.js / React** frontends but transport-agnostic.

---

## 2. Core Concepts

### 2.1 Game State

- **Single immutable object** (the "big game object").
- Updated via **reducers**, with Immer patches generated for syncing.
- Example:

```tsx
interface GameState {
  players: Record<string, { hand: string[]; score: number }>;
  deck: string[];
  discardPile: string[];
  turn: string; // playerId
  phase: string; // e.g., "main", "discard"
}

```

---

### 2.2 Actions

- Actions are **atomic changes** initiated by players.
- Defined with `{ schema, authorize, reducer }`.
- **Zod schema** ensures strong type inference.

**Example: Play Card**

```tsx
const playCard = defineAction({
  schema: z.object({ cardId: z.string() }),
  authorize: (ctx, { cardId }, state) => ctx.playerId === state.turn,
  reducer: (state, { cardId }, ctx) => {
    const hand = state.players[ctx.playerId].hand
    if (!hand.includes(cardId)) return
    hand.splice(hand.indexOf(cardId), 1)
    state.discardPile.push(cardId)
    ctx.log(`${ctx.playerId} played card ${cardId}`)
  }
})

```

- `authorize` ensures **active player only** (default).
- Optional for more complex logic (e.g., simultaneous discard phase).

---

### 2.3 Masking

- Each player sees a **personalized view** of the game state.
- Masking function example:

```tsx
mask: (state, playerId) => ({
  ...state,
  players: Object.fromEntries(
    Object.entries(state.players).map(([id, p]) => [
      id,
      {
        ...p,
        hand: id === playerId ? p.hand : Array(p.hand.length).fill(null)
      }
    ])
  )
})

```

- Ensures hands of other players are hidden while server retains full state.

---

### 2.4 Phases

- **Named stages** controlling allowed actions and flow.
- Example:

```tsx
phases: {
  main: {
    available: ctx => ctx.activePlayerOnly,
    actions: ["playCard", "pass"],
    onEnd: (state) => checkVictory(state)
  },
  discard: {
    simultaneous: true,
    available: ctx => ctx.allPlayers,
    actions: ["discardCard"],
    endWhen: state => state.players.every(p => p.hasDiscarded)
  }
}

```

- `main` phase → classic turn-based flow.
- `discard` phase → all players act simultaneously.

---

### 2.5 Turn Handling

- Active player → `ctx.activePlayerOnly`.
- Nested turns via **phase stack**:

```tsx
ctx.pushPhase({
  name: "discardPhase",
  simultaneous: true,
  actions: ["discardCard"],
  endWhen: state => state.players.every(p => p.hasDiscarded)
})

```

- When completed, engine **pops phase** and resumes original turn.

---

### 2.6 Simultaneous Moves

- Example: each opponent discards a card mid-turn:

```tsx
const discardCard = defineAction({
  schema: z.object({ cardId: z.string() }),
  authorize: (ctx, payload, state) => ctx.phase === "discard" && state.players[ctx.playerId].hand.includes(payload.cardId),
  reducer: (state, { cardId }, ctx) => {
    state.players[ctx.playerId].hand = state.players[ctx.playerId].hand.filter(c => c !== cardId)
    state.players[ctx.playerId].hasDiscarded = true
  }
})

```

- Engine waits until all required players have discarded before continuing.

---

### 2.7 Context & DSL

**DSL primitives:**

- `turn({ order })` → classic turn-based flow.
- `phase({ name, sequence, onEnd })` → named game stages.
- `simultaneous({ players, actions?, onComplete })` → multiple players act concurrently.

**Flow example:**

```tsx
defineGame({
  initialState: () => ({ ... }),
  context: turn({ order: state => Object.keys(state.players) }),
  phases: {
    main: { ... },
    discard: { ... }
  }
})

```

- Context stack supports **nested phases** and **interrupts**.

---

### 2.8 RNG

- Seeded per game (`ctx.rng`).
- Deterministic results for replay and debugging.

```tsx
const drawCard = defineAction({
  schema: z.object({}),
  reducer: (state, payload, ctx) => {
    const index = Math.floor(ctx.rng() * state.deck.length)
    const card = state.deck.splice(index, 1)[0]
    state.players[ctx.playerId].hand.push(card)
  }
})

```

---

### 2.9 Persistence & Sync

- Snapshots + Immer patches stored in **Postgres**.
- Action log for **time travel & replay**.
- Clients sync via:
    1. Snapshot on join/reconnect.
    2. Incremental patches.
    3. Deduplication via **idempotent action IDs**.

---

### 2.10 Developer Experience

- **React hooks**: `useGameState(gameId, gameDefinition)` → returns typed state, `sendAction`.
- **Inspector UI**: visualizes state, logs, patch history.
- **Unit testing**: Vitest per reducer + snapshot tests.

---

## 3. Example Game: "Mini Card Game"

```tsx
const MiniCardGame = defineGame({
  initialState: () => ({
    players: { p1: { hand: ["A", "B"], hasDiscarded: false }, p2: { hand: ["C", "D"], hasDiscarded: false } },
    discardPile: [],
    turn: "p1",
    phase: "main"
  }),
  mask: (state, playerId) => ({ ...state, players: Object.fromEntries(
    Object.entries(state.players).map(([id, p]) => [id, { ...p, hand: id === playerId ? p.hand : Array(p.hand.length).fill(null) }])
  )}),
  actions: { playCard, discardCard, pass },
  context: turn({ order: state => Object.keys(state.players) }),
  phases: {
    main: { available: ctx => ctx.activePlayerOnly, actions: ["playCard", "pass"] },
    discard: { simultaneous: true, available: ctx => ctx.allPlayers, actions: ["discardCard"], endWhen: state => Object.values(state.players).every(p => p.hasDiscarded) }
  }
})

```

**Flow:**

1. Player p1 plays a card.
2. Triggers `discard` phase → p2 discards simultaneously.
3. When both have discarded, `discard` phase ends → p1 continues turn.

---

## 4. Networking & Transport

- WebSocket adapter by default.
- Transport-agnostic interface for future SSE/WebRTC.
- Optimistic updates supported.
- Reconnect: client sends last known version → server sends snapshot + missing patches.

---

## 5. Roadmap

1. **MVP**:
    - State + Immer, actions + schemas, masking.
    - Context DSL: turns, phases, simultaneous.
    - WebSocket transport, Postgres persistence.
2. **Phase 2**:
    - Inspector + debugging, time travel.
    - Nested phases & interrupts.
3. **Phase 3**:
    - Plugins: turn manager helpers, RNG policies, auth adapters.
    - Advanced transports (WebRTC).

---

## 6. Non-Goals

- AI opponents (v1).
- Money/market features.
- Complex clustering/leader election.

---

## 7. License

- **MIT**

[Implementation Plan](https://www.notion.so/Implementation-Plan-2526519d3fb880899f84d73c75ddee41?pvs=21)