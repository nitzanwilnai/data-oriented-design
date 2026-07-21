# DOD Patterns Reference (language-agnostic)

Core patterns from *High Performance Unity Game Development* (*Using data-oriented
design*) by Nitzan Wilnai, written language-neutrally. Code is pseudocode close to
JS; for the concrete HTML/JS spelling (TypedArrays, Canvas, rAF) see `html-js.md`.

## Contents
1. Structure of Arrays vs Array of Structs
2. Pure Logic Function Anatomy
3. Array-as-List / Stack / Queue
4. Object / Resource Pool
5. Index-Based Identity
6. Data-First Feature Flow
7. Indices Instead of Dictionaries
8. Separating Live/Dead to Remove Branches

---

## 1. Structure of Arrays vs Array of Structs

### Array of Structs (avoid)
```
entities = [ {x, y, vx, vy, hp}, {x, y, vx, vy, hp}, ... ]
```
Each element is a separate heap object. Iterating to move everything loads `hp` and
every other field into cache even though movement only needs `x,y,vx,vy`. Memory is
scattered; every access risks a cache miss (and in managed languages the numbers are
boxed).

### Structure of Arrays (use)
```
posX[], posY[], velX[], velY[], hp[]
```
Iterating `posX/posY/velX/velY` streams exactly those fields, contiguous, cache-line
dense. `hp` isn't touched so it doesn't evict useful data. This is the single most
important DOD pattern.

---

## 2. Pure Logic Function Anatomy

Every logic function is **Data-In → Transformation → Data-Out**, takes plain data,
mutates it in place or returns a primitive, and allocates nothing in the hot path.

```
// Data-In: state (to transform), config, dt
// Data-Out: state.posX/posY modified in place
function moveEnemies(state, config, dt) {
  for (let i = 0; i < state.enemyCount; i++) {
    // toward origin: dir = -normalize(pos)
    let x = state.posX[i], y = state.posY[i];
    let inv = config.enemyVel * dt / Math.sqrt(x*x + y*y);
    state.posX[i] -= x * inv;
    state.posY[i] -= y * inv;
  }
}
```

Secondary outputs are returned as primitives or written into existing state fields —
never as a freshly allocated result object per call:
```
function tick(state, config, dt) {
  state.time += dt;
  moveEnemies(state, config, dt);
  return checkGameOver(state, config);   // bool, no allocation
}
```

---

## 3. Array-as-List / Stack / Queue

Pre-size arrays to a maximum; track a `count`. No growable list type in hot paths.

**List add / remove-by-value (preserves order):**
```
// add
arr[count++] = value;

// remove first matching value, shifting left
let w = 0;
for (let i = 0; i < count; i++) if (arr[i] !== value) arr[w++] = arr[i];
count = w;
```

**Stack (O(1) add/remove; use when order doesn't matter — spawn pools):**
```
arr[count++] = value;          // push
const value = arr[--count];    // pop
```

**Swap-remove (O(1) remove by index; order-independent):**
```
arr[i] = arr[--count];         // move last into slot i
```

Why not a growable list? It allocates and can trigger the collector; array + count
gives zero allocation and full control. Don't wrap these one-liners in a helper class
— inline is clearer and the right solution differs per case.

---

## 4. Object / Resource Pool

Pre-create the maximum number of presentation resources once; show/hide by index;
never create/destroy during gameplay.

```
// init (once)
pool = new Array(config.maxSprites);
for (let i = 0; i < config.maxSprites; i++) { pool[i] = makeSprite(); hide(pool[i]); }

// spawn = activate slot i (i comes from a stack pop of free slots)
show(pool[i]); place(pool[i], state.posX[i], state.posY[i]);

// despawn = deactivate
hide(pool[i]);
```

Pick a maximum up front; only *add* (activate) at runtime. Freeing/shrinking at
runtime fragments memory and (in managed runtimes) feeds the collector.

---

## 5. Index-Based Identity

An entity is an integer index into every parallel array — no per-entity object.
"Ball 3" means `ballX[3], ballY[3], ...`. This is what makes SoA, pooling, and
save/load uniform: everything keys off `i`.

```
function spawnBall(state, x, y, vy) {
  const i = state.ballCount++;
  state.ballX[i] = x; state.ballY[i] = y;
  state.ballVX[i] = 0; state.ballVY[i] = vy;
  return i;
}
```

---

## 6. Data-First Feature Flow

Adding a feature is always the same steps, so its cost is constant:

1. Data needed? → add to config (static) or state (dynamic).
2. Allocated when? → add arrays to the one allocation step, sized from config.
3. Transformed by? → add pure function(s) to logic.
4. Initialized how? → set up in start-run.
5. Runs per frame? → call from tick.
6. Shown how? → sync visuals from state in the board.
7. Designer knob? → add to config + its author-time validation.
8. Saved? → extend save/load in fixed field order; bump version.

No existing system is refactored — you only add data and logic.

---

## 7. Indices Instead of Dictionaries

Hash lookups are far slower than an array index. Assign integer IDs at author/build
time; index arrays at runtime.

```
// author/build time: each type gets a stable integer id; config holds parallel
// arrays indexed by that id.
config.typeVel    = Float32Array([...]);   // indexed by typeId
config.typeRadius = Float32Array([...]);

// state stores the type id per entity slot; runtime uses a plain index — no map.
state.typeId = Int32Array(config.maxEntities);
const v = config.typeVel[state.typeId[i]];
```

The only acceptable dictionary is a **one-time load-time remap** (saved stable
id/name → current index), built once and discarded. Never during a frame.

---

## 8. Separating Live/Dead to Remove Branches

Instead of an `isAlive` flag tested every iteration, keep two index arrays. Loops over
the live list have no per-item branch, and spawn/despawn is a stack move between lists.

```
// state
liveIdx[], liveCount
deadIdx[], deadCount

// spawn: pop dead, push live
const i = deadIdx[--deadCount];
liveIdx[liveCount++] = i;

// despawn: pop from live, push dead (swap-remove from live)
// (find position p of i in liveIdx, then:)
liveIdx[p] = liveIdx[--liveCount];
deadIdx[deadCount++] = i;

// update loop — no isAlive check:
for (let k = 0; k < liveCount; k++) { const i = liveIdx[k]; /* update i */ }
```
