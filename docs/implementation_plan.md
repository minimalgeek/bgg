# **Monorepo Folder Structure**

```
/monorepo
  /packages
    /core
      src/
        index.ts             # exports
        game.ts              # defineGame, phase/turn DSL
        actions.ts           # defineAction, schema + authorize + reducer
        types.ts             # GameState, ActionContext, Phase, Turn types
        masking.ts           # mask(state, playerId)
      package.json
      tsconfig.json
    /server
      src/
        index.ts
        gameServer.ts
        wsAdapter.ts
        postgresAdapter.ts
      package.json
      tsconfig.json
    /client
      src/
        index.ts
        useGame.ts           # React hook
        types.ts             # typed GameState, Actions
      package.json
      tsconfig.json
    /inspector
      src/
        index.ts
        inspectorUI.ts
      package.json
      tsconfig.json
  /apps
    /nextjs-app
      pages/
      package.json
      tsconfig.json
  /games
    /MiniCardGame
      src/
        index.ts             # defines the game using core
      package.json
      tsconfig.json
    /TicTacToe
      src/
        index.ts
      package.json
      tsconfig.json
  package.json
  turbo.json
  tsconfig.base.json

```

---

# **Implementation Plan (Step 1 – Core)**

### Goal

- Build `core` library with **immutable game state**, **actions**, **DSL for turns/phases/simultaneous moves**, and **masking**.

### Step-by-Step

1. Define `GameState` and generic types in `types.ts`.
2. Implement `defineAction` in `actions.ts`:
    - Validate payload using Zod.
    - Authorize function.
    - Reducer with ImmerJS patch generation.
3. Implement `mask` system in `masking.ts`.
4. Implement `defineGame` in `game.ts`:
    - Phases, turn stack, simultaneous moves.
    - Context stack for nested phases.
5. Unit tests for:
    - Reducers
    - Patch generation
    - DSL correctness

---

### TypeScript Signatures (Skeleton)

**types.ts**

```tsx
export type PlayerId = string

export interface GameState {
  turn: PlayerId
  phase: string
  players: Record<PlayerId, any>
  [key: string]: any
}

export interface ActionContext<S extends GameState> {
  playerId: PlayerId
  activePlayerId: PlayerId
  phase: string
  rng: () => number
  state: S
}

export type Reducer<S extends GameState, P> = (state: S, payload: P, ctx: ActionContext<S>) => void
export type Authorize<S extends GameState, P> = (ctx: ActionContext<S>, payload: P) => boolean

```

**actions.ts**

```tsx
import { z } from "zod"
import { Reducer, Authorize, ActionContext } from "./types"

export interface GameAction<S, P> {
  name: string
  schema: z.ZodType<P>
  authorize: Authorize<S, P>
  reducer: Reducer<S, P>
}

export function defineAction<S, P>(action: GameAction<S, P>): GameAction<S, P> {
  return action
}

```

**masking.ts**

```tsx
import { GameState, PlayerId } from "./types"

export type MaskFunction<S extends GameState> = (state: S, playerId: PlayerId) => S

export const defaultMask: MaskFunction<GameState> = (state, playerId) => state

```

**game.ts**

```tsx
import { GameAction, MaskFunction } from "./actions"
import { GameState } from "./types"

export interface Phase<S extends GameState> {
  name: string
  actions?: string[]
  simultaneous?: boolean
  available?: (ctx: any) => boolean
  endWhen?: (state: S) => boolean
}

export interface DefineGameOptions<S extends GameState> {
  initialState: () => S
  actions: Record<string, GameAction<S, any>>
  mask: MaskFunction<S>
  phases?: Record<string, Phase<S>>
}

export function defineGame<S extends GameState>(options: DefineGameOptions<S>) {
  return options
}

```

---

This sets up **Step 1**: the core library with **types, actions, game definition, and masking**.

Next, Step 2 will **implement the DSL fully**: turns, nested phases, simultaneous moves, and action availability calculation.