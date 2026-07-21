---
name: data-oriented-design
description: >
  Language-agnostic architecture guide for building games and performance-sensitive
  apps with Data-Oriented Design (DOD) — adapted from *High Performance Unity Game
  Development (Using data-oriented design)* by Nitzan Wilnai, generalized to any
  language and with a first-class HTML5/JavaScript/Canvas track. Use this skill
  whenever building or reviewing a game (or real-time/simulation code) in DOD style
  outside Unity — especially HTML/JS/Canvas, but also C, C++, Rust, Go, Zig, or
  plain C#. Trigger when the user asks to: scaffold a game or feature in DOD style,
  generate DOD-compliant code, review code for DOD violations, or explain DOD
  concepts. Also trigger on: "data-oriented design", "DOD", "parallel arrays",
  "structure of arrays", "SoA vs AoS", "TypedArray", "Float32Array", "cache
  locality", "data-in/transformation/data-out", "pure logic functions", "separate
  data from logic", "array + count", "index-based identity", "object pool",
  "avoiding dictionaries", "branchless", "avoid per-frame allocation", "GC
  pressure", "Balance / GameData / Logic / Board / Game".
---

# Data-Oriented Design — Language-Agnostic Architecture Skill

This skill encodes the DOD architecture from *High Performance Unity Game Development*
(*Using data-oriented design*) by Nitzan Wilnai (Manning), generalized so it applies
in **any language**. It has a first-class **HTML5 / JavaScript / Canvas** track,
because the memory-layout ideas need a specific translation there (TypedArrays, GC
avoidance).

When helping build or review a DOD project, follow these patterns and terminology
precisely. For Unity specifically, prefer the `unity-dod-architecture` skill; use
this one for HTML/JS and every other language.

---

## Core Philosophy (language-independent)

DOD is not a framework, an ECS, or a library. It is a coding paradigm built on three
pillars:

1. **Performance** — organize data in contiguous arrays for cache locality; separate
   data from logic so the CPU streams exactly the fields it needs.
2. **Reduced Complexity** — pure logic functions; clear data ownership; no
   abstraction for its own sake.
3. **Extensibility** — adding a feature is only "what new data + what new logic";
   the cost of a feature stays constant and existing systems don't get refactored.

**The DOD Mantra:** every logic function takes **data in**, **transforms** it, and
writes **data out**. Logic functions do not reach back into the presentation layer or
the game loop. They are pure transformations over plain data.

This holds regardless of language. What *changes* per language is the mechanism for
getting contiguous memory and avoiding allocation stalls — see the language notes
below and `references/html-js.md`.

---

## The Five Architectural Components

The book's five roles map onto any language. Names in parentheses are the generic /
HTML equivalents.

```
Balance   — static config data; set at author/build time; never changes at runtime.
            (a.k.a. "config", "tuning", "constants")
GameData  — mutable runtime state, held as parallel arrays; only Logic modifies it.
            (a.k.a. "world", "state")
Logic     — pure/free functions; the ONLY place GameData is modified.
            (a module of stand-alone functions — no methods, no `this`)
Board     — the presentation + input layer; reads GameData to render, writes input
            into GameData, calls Logic. (a.k.a. "view", "renderer+input")
Game      — the top-level loop/orchestrator; owns Balance + GameData; drives Board.
            (a.k.a. "main", the requestAnimationFrame / main-loop owner)
```

Supporting roles (create only when the game needs them):
- **MetaData** — data that persists across sessions (best score, settings). Separate
  from GameData; saved/loaded on its own.
- **Assets** — a single place that owns loaded images/audio/prefabs, handed out by
  dedicated getters. Preload; never load in the hot loop.
- **Save/Load** — plain functions that serialize GameData / MetaData in a fixed field
  order, versioned. (`GameDataIO` / `MetaDataIO` in the book.)

The dependency arrow only points one way: **Game → Board → Logic → (Balance,
GameData)**. Logic never calls Board or Game. Board never mutates GameData except by
calling Logic (input is the one allowed exception — it may stage raw input into
GameData for Logic to consume).

---

## Data Rules

### Balance (config)
- Plain data set at author/build time. **Never modified at runtime.**
- Holds counts, sizes, radii, velocities, spawn rates, score tables, curves, rules.
- Validate it at **author/build time**, not at runtime (see "Validate at author
  time"). Runtime code trusts Balance completely — zero defensive checks.
- Arrays in Balance are fine — their size is known once Balance is loaded.

### GameData (state)
- All mutable runtime state: positions, velocities, levels, counts, indices, timers,
  flags, current menu/phase enum.
- **Structure of Arrays, not Array of Structs.** Store one array per field, indexed
  by entity slot — never an array of per-entity objects:
  ```
  // CORRECT — parallel arrays (Structure of Arrays)
  ballX[], ballY[], ballVX[], ballVY[]

  // WRONG — array of objects (Array of Structs); scatters memory, boxes numbers
  balls[] where each ball = { x, y, vx, vy }
  ```
- **Identity is an index.** An entity is just an integer `i` that indexes every
  parallel array. No per-entity object, no pointer, no class instance.
- **Array + count instead of a dynamic list.** Pre-size arrays to a maximum; track a
  live `count`. Add with `arr[count++] = v`; O(1) remove-from-stack with
  `v = arr[--count]`.
- **Separate live/dead index arrays** instead of an `isAlive` flag you test every
  iteration (this also removes a per-item branch — see Branching).
- Boolean flags and phase/menu enums live in GameData.
- Allocate all arrays **once** (at load or level start), sized from Balance. Never
  allocate during the frame loop.

### MetaData
- Cross-session data (best score, unlocks, options). Saved/loaded separately from
  GameData.

---

## Logic Rules

Logic is a module of stand-alone functions — no class holding state, no `this`.

- Every function is **Data-In → Transformation → Data-Out**. It takes `gameData` and
  `balance` (and `dt`, `metaData`, etc. as needed) and mutates `gameData` in place
  and/or returns a value.
- Public entry points (`AllocateGameData`, `StartGame`, `Tick`) are the surface the
  Board calls. Everything else is a private helper in the same module.
- Logic **never** references the Board, the Game loop, the DOM, the canvas, timers,
  or any I/O. It only touches plain data.
- Logic never uses the frame callback itself, coroutines, promises, async, or
  event handlers. It is driven synchronously from the loop.
- **Allocate nothing in the hot path.** No new arrays/objects/closures inside `Tick`
  or anything it calls every frame. Use pre-allocated scratch buffers in GameData.
- Prefer squared distance to real distance for comparisons (skip the sqrt).
- Avoid early returns in tick-level functions — keep the whole execution path visible
  and run it fully; decide outcomes at the end.

### The Tick pattern
```
function Tick(gameData, balance, dt) {
  gameData.time += dt;
  spawnIfNeeded(gameData, balance);
  moveEntities(gameData, balance, dt);
  collide(gameData, balance);
  const gameOver = checkGameOver(gameData, balance);
  return gameOver;   // or write into gameData; Board reads it after Tick
}
```

---

## Board Rules (presentation + input)

The Board is the only place that knows about the screen and the input device (canvas,
DOM, terminal, whatever). In HTML that means the `<canvas>` 2D/WebGL context and
pointer/key events.

- Board **calls Logic**; Logic never calls Board.
- Each frame: read input into GameData → call `Logic.Tick(...)` → render the current
  GameData → then (last) react to end-of-frame results (game over, level change).
- Board **owns pools** of presentation resources (sprites, DOM nodes, audio voices).
  Pre-create them; show/hide by index; never create/destroy per frame.
- Board reads visual positions **from GameData after** Logic ran — it does not compute
  gameplay itself.
- Board does not own Balance or GameData — it receives them as parameters.

## Game (loop) Rules

- Game owns Balance, GameData, MetaData and drives the Board. There is one loop; the
  Board does not run its own.
- Game owns phase/menu transitions (show/hide Board vs menus).
- Load Balance, allocate GameData, load saves, init Board — once, at startup.

---

## Memory & Allocation Rules

The #1 rule everywhere: **allocate at load/level-start, never during gameplay.**

- Pre-allocate every array from Balance sizes in one `AllocateGameData` step.
- Reuse scratch buffers held in GameData for temporary per-frame work.
- Use **pools**: pre-create presentation objects; activate/deactivate by index.
- Use **array + count**, not a growable list, for add/remove during play.
- Pick a **maximum** pool size up front; only *add* at runtime. Freeing/shrinking at
  runtime fragments memory and (in managed languages) feeds the collector.

Why this matters is language-specific:
- **C / C++ / Rust / Zig:** contiguous arrays = cache hits; per-frame `malloc`/`new`
  = stalls and fragmentation.
- **Managed languages (JS, C#, Go, Java):** per-frame allocation feeds the garbage
  collector, and a GC pause is a dropped frame. In DOD you allocate up front so the
  collector has nothing to do during play. **A GC spike is the JS/C# equivalent of a
  cache-miss stall — and it's caused by allocation in the loop.**

---

## Structure of Arrays by Language

The *concept* (SoA + index identity + no per-frame allocation) is universal. The
*mechanism* differs:

| Language | How you get contiguous, un-boxed parallel arrays |
|---|---|
| **JavaScript / HTML** | **TypedArrays** — `Float32Array`, `Int32Array`, `Uint8Array`, etc. These are the whole game. A plain `[]` of numbers is boxed and pointer-chased; a `Float32Array` is contiguous C-style memory. See `references/html-js.md`. |
| **C / C++** | Plain arrays / `std::vector<T>` of primitives, one per field. Struct-of-arrays, not array-of-structs. |
| **Rust** | `Vec<f32>` per field, index = entity id; a pure-logic crate with no `wgpu`/`winit` deps, consumed by a native render crate. See `references/rust.md`. |
| **C#** (non-Unity) | Arrays of primitives / `struct` (blittable). Same as the book. |
| **Go** | Slices of primitives, one per field. Avoid slices of pointers. |
| **Python / high-level** | `numpy` arrays per field when it's hot; accept that pure-Python loops won't be cache-optimal — DOD still helps clarity and vectorization. |

**For HTML/JS specifically, read `references/html-js.md` before writing code.** It
covers TypedArrays, a Canvas Board, a `requestAnimationFrame` Game loop, and the
per-frame-allocation traps unique to JS (closures, `.map/.filter`, spread,
destructuring, string building, `Math.hypot` temporaries).

---

## Validate at Author/Build Time

Push all validation to when config/content is authored or built, so runtime code
carries zero defensive checks.

- Build/author step checks Balance (counts > 0, within caps, curves monotonic, IDs
  unique) and fails loudly if wrong.
- Runtime trusts the data: `state.x = new Float32Array(balance.maxBalls)` with no
  guard. A reader never has to wonder "can this be negative?"
- Only add a validation check when a real bug shows up in development — don't
  pre-empt every hypothetical.

---

## Indices Instead of Dictionaries

Hash-map lookups (`Map`, `{}` as a dict, `Dictionary`) are far slower than an array
index (hashing + collision chains + pointer chasing + poor cache behavior). Assign
integer IDs at author/build time and use them as array indices at runtime.

- Author one record per type; the build step assigns each a stable integer ID and
  flattens type data into Balance arrays indexed by that ID.
- GameData stores the type ID per entity slot; look up per-type config with a direct
  array index — no map.
- The **only** acceptable dictionary is a **one-time remap at load** (saved stable
  string/ID → current index), built once and discarded. Never used during a frame.

---

## Branching & Branchless Thinking

Branches cost performance (misprediction flushes the pipeline) and add complexity.
Reduce them by **reorganizing data**, not by cleverer conditionals.

- Decide once, up front, or at author time — not inside the hot loop.
- **Separate data instead of testing it:** keep live and dead entities in separate
  index arrays so loops never test an `isAlive` flag per item.
- Avoid early returns in tick functions — they hide branches and risk skipping
  end-of-frame work (visual sync, deallocation).
- Watch for hidden branches: default parameters, optional chaining in hot loops, and
  (in some engines) expensive null/liveness checks.

---

## Data-First Feature Development Flow

Adding any feature always follows the same sequence — this is why feature cost stays
constant:

1. **What data does it need?** → add fields to Balance (static) or GameData (dynamic).
2. **When is it allocated?** → add arrays to `AllocateGameData` (sized from Balance).
3. **What transforms it?** → add pure function(s) to Logic.
4. **How does it start?** → initialize in `StartGame`.
5. **Does it run per frame?** → call it from `Tick`.
6. **How is it shown?** → sync visuals from GameData in the Board.
7. **Needs a designer knob?** → add to Balance + its author-time validation.
8. **Needs saving?** → extend Save/Load in fixed field order, bump the version.

No existing system gets refactored. You only add data and logic.

---

## Common Anti-Patterns to Flag

| Anti-Pattern | DOD Correction |
|---|---|
| Array of per-entity objects (`balls[i].x`) | Parallel arrays / TypedArrays (`ballX[i]`); identity is the index |
| Classes that hold data *and* methods that mutate it | Split into plain data + pure Logic functions |
| Methods on entities (`ball.update()`) | Free function over arrays (`moveBalls(gameData, balance, dt)`) |
| Growable list (`push`/`splice`, `List<T>`) in the loop | Pre-sized array + `count`; stack add/remove |
| `new` / object literals / closures created every frame | Allocate once up front; reuse scratch buffers (GC pause = dropped frame) |
| `.map`/`.filter`/`.reduce`/spread in the hot path (JS) | Explicit `for` loops writing into pre-allocated arrays |
| `Map`/`{}`/`Dictionary` lookups during gameplay | Integer IDs assigned at author time; array indexing; map only at load |
| `isAlive` flag tested every iteration | Separate live/dead index arrays — the branch disappears |
| Inheritance hierarchies for entity types | Type ID indexing Balance arrays; all entities share the same arrays |
| Generic helper wrappers around array add/remove | Inline the one line — explicit beats abstracted here |
| Defensive runtime checks on config | Validate at author/build time; trust data at runtime |
| Logic that reads the canvas/DOM/input directly | Logic touches only plain data; Board owns presentation + input |
| Loading assets / building strings in the loop | Preload assets; build display strings only when the value changes |
| Early return in a tick function | Run the full path; act on results at the end |
| Reaching for ECS to "do DOD" | ECS is one implementation of DOD, not a requirement; parallel arrays already are DOD |

---

## File / Module Layout (generic)

| Module | Purpose |
|---|---|
| `balance.*` | Config data + author-time validation. |
| `gameData.*` | GameData: parallel arrays + counts + flags/enums. |
| `logic.*` | Pure functions; the only place GameData changes. Split by domain when large (`ballLogic`, `gridLogic`, `collisionLogic`). |
| `board.*` | Presentation + input (canvas render + pointer/key handling). |
| `game.*` / `main.*` | Loop owner; holds Balance + GameData; drives Board. |
| `metaData.*`, `saveLoad.*` | Optional: persistence. |

For the concrete HTML/JS spelling of every one of these (with TypedArrays, a Canvas
Board, and a rAF loop), see `references/html-js.md`.

---

## For deeper reference, see:
- `references/html-js.md` — the HTML5/JavaScript/Canvas track: TypedArrays, Canvas
  Board, rAF Game loop, and JS-specific per-frame-allocation traps. **Read this
  before writing HTML/JS DOD code.**
- `references/rust.md` — the Rust track: parallel `Vec`s vs `#[repr(C)]`/`Pod` GPU
  instance structs, a pure-logic-crate + wgpu/winit-native-crate workspace split, the
  read-only-UI/`UiAction` enforcement pattern, pre-allocated scratch buffers for
  instanced rendering, `cargo test`-based golden/parity testing, and Rust-specific
  per-frame-allocation traps — grounded in two real shipped Rust DOD games. **Read
  this before writing native Rust DOD code.**
- `references/patterns.md` — language-agnostic patterns (SoA, pure logic anatomy,
  array-as-list/stack, index identity, pooling, indices-not-dictionaries).
- `references/anti-patterns.md` — OOP habits → DOD corrections, with generic +
  JS examples.
