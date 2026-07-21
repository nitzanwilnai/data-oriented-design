# DOD in HTML5 / JavaScript / Canvas

The concrete translation of the DOD architecture to a browser game. Read this before
writing HTML/JS DOD code. Nothing here needs a framework or a build step — plain ES
modules, a `<canvas>`, and `requestAnimationFrame`.

The two ideas that make DOD *real* in JavaScript:

1. **TypedArrays are the whole game.** A plain `[]` of numbers stores boxed values
   scattered on the heap and pointer-chased on every access. A `Float32Array` /
   `Int32Array` / `Uint8Array` is one contiguous block of C-style memory — no boxing,
   cache-friendly, and it can't silently grow. Every parallel gameplay array should
   be a TypedArray sized once from Balance.
2. **A GC pause is a dropped frame.** JavaScript is garbage-collected; you can't free
   memory, you can only avoid allocating. Every `new`, object literal, closure,
   temporary array, or built string *inside the frame loop* feeds the collector, and
   when it runs you drop a frame. DOD's "allocate up front, never in the loop" rule is
   how you keep the collector idle during play. **Allocation in the loop is the JS
   analog of a cache-miss stall.**

---

## 1. Structure of Arrays with TypedArrays

### Array of objects (avoid)
```js
// Each ball is a heap object; numbers are boxed (float64 in an object slot);
// iterating balls pointer-chases and loads fields you don't need.
const balls = [];
balls.push({ x, y, vx, vy, r });
for (const b of balls) { b.vy += g * dt; b.x += b.vx * dt; b.y += b.vy * dt; }
```

### Structure of arrays with TypedArrays (use)
```js
// gameData.js — allocate ONCE, sized from balance. Identity is the index i.
export function allocateGameData(gd, balance) {
  const n = balance.maxBalls;
  gd.ballX  = new Float32Array(n);
  gd.ballY  = new Float32Array(n);
  gd.ballVX = new Float32Array(n);
  gd.ballVY = new Float32Array(n);
  gd.ballCount = 0;               // array + count, not a growable list
}

// logic.js — a pure function over the arrays; zero allocation.
export function moveBalls(gd, balance, dt) {
  for (let i = 0; i < gd.ballCount; i++) {
    gd.ballVY[i] += balance.gravity * dt;
    gd.ballX[i]  += gd.ballVX[i] * dt;
    gd.ballY[i]  += gd.ballVY[i] * dt;
  }
}
```

**Choosing the width:** `Float32Array` for positions/velocities (plenty of precision
for gameplay, half the memory of float64, better cache density). `Int32Array` for
counts/ids that can be large or negative. `Uint8Array` for small enums like a tile
color `0..3` or level `1..5` — a full 8×8 grid of colors is 64 bytes.

**A 2D grid** is a flat TypedArray, indexed `row * width + col`:
```js
gd.gridColor = new Uint8Array(balance.gridW * balance.gridH); // 0..numColors-1
gd.gridLevel = new Uint8Array(balance.gridW * balance.gridH); // 0 = empty, else 1..maxLevel
const idx = r * balance.gridW + c;
```
Use `level === 0` to mean "empty cell" so you don't need a parallel occupancy array.

---

## 2. The module layout

```
index.html        <script type="module" src="src/main.js">
src/balance.js     config data + author-time validate()
src/gameData.js    allocateGameData(gd, balance): parallel TypedArrays + counts + flags
src/logic.js       pure functions; the ONLY place gameData changes (split when large)
src/board.js       canvas render + pointer/key input; calls logic
src/main.js        the Game loop: owns balance+gameData, drives board via rAF
```

`gameData` is a plain object whose *properties are TypedArrays*. It carries no methods.
`logic` functions are `export function name(gd, balance, ...)` — never methods, never
`this`. `board` and `main` are the only modules that touch the DOM/canvas/events.

---

## 3. Balance + author-time validation

```js
// balance.js
export const balance = {
  gridW: 8, gridH: 8, numColors: 4, maxLevel: 5,
  movesPerLevel: 5, ballsPerLevel: 5, maxBalls: 1,
  gravity: 900, restitution: 0.72, ballRadius: 8, pegRadius: 16,
  scoreByLevel: new Int32Array([0, 10, 25, 60, 150, 400]), // index by level; [0] unused
  // ...curves, etc.
};

// Author-time check — call once at startup in dev, or in a build script. Runtime
// gameplay code then trusts balance completely and adds zero defensive checks.
export function validate(b) {
  const errs = [];
  if (b.numColors < 3) errs.push('numColors must be >= 3');
  if (b.maxLevel < 2) errs.push('maxLevel must be >= 2');
  if (b.scoreByLevel.length < b.maxLevel + 1) errs.push('scoreByLevel too short');
  if (errs.length) throw new Error('balance invalid:\n' + errs.join('\n'));
}
```

---

## 4. Logic: pure, allocation-free, index-based

```js
// logic.js — every function is data-in -> transform -> data-out.
// No DOM, no canvas, no events, no timers, no `new` in the hot path.

export function stepBall(gd, balance, i, dt, bounds) {
  gd.ballVY[i] += balance.gravity * dt;
  gd.ballX[i]  += gd.ballVX[i] * dt;
  gd.ballY[i]  += gd.ballVY[i] * dt;

  // walls: reflect vx, damp by restitution
  if (gd.ballX[i] - balance.ballRadius < bounds.left) {
    gd.ballX[i] = bounds.left + balance.ballRadius;
    gd.ballVX[i] = -gd.ballVX[i] * balance.restitution;
  } else if (gd.ballX[i] + balance.ballRadius > bounds.right) {
    gd.ballX[i] = bounds.right - balance.ballRadius;
    gd.ballVX[i] = -gd.ballVX[i] * balance.restitution;
  }
}
```

**Scratch buffers, not temporaries.** When a function needs a working array (a visited
mask for flood-fill, a marks grid for match detection), allocate it once in
`allocateGameData` and clear it in place — never `new` it inside the function:

```js
// allocateGameData:
gd.scratchMarks = new Uint8Array(balance.gridW * balance.gridH);

// in logic, reuse it:
gd.scratchMarks.fill(0);   // clear, no allocation
```

**Return primitives or write into gameData** for secondary outputs — don't allocate a
result object every call. If you must report a small fixed set (e.g. counts), write
them into pre-existing `gd` fields.

---

## 5. Board: Canvas render + input

The Board is the only module that touches the canvas and events. It reads gameData to
draw and writes input into gameData; it calls logic.

```js
// board.js
export function render(ctx, gd, balance, view) {
  ctx.clearRect(0, 0, view.w, view.h);
  for (let r = 0; r < balance.gridH; r++) {
    for (let c = 0; c < balance.gridW; c++) {
      const idx = r * balance.gridW + c;
      const level = gd.gridLevel[idx];
      if (level === 0) continue;                 // empty
      const x = view.ox + c * view.cell + view.cell / 2;
      const y = view.oy + r * view.cell + view.cell / 2;
      ctx.beginPath();
      ctx.arc(x, y, balance.pegRadius * (0.7 + 0.09 * level), 0, TAU);
      ctx.fillStyle = view.colorCss[gd.gridColor[idx]]; // colorCss precomputed, not built here
      ctx.fill();
    }
  }
  for (let i = 0; i < gd.ballCount; i++) {
    ctx.beginPath();
    ctx.arc(gd.ballX[i], gd.ballY[i], balance.ballRadius, 0, TAU);
    ctx.fill();
  }
}
const TAU = Math.PI * 2;
```

**Input pools/handlers are attached once**, not per frame. Handlers write raw intent
into gameData (or call a logic entry point); they don't run gameplay themselves:

```js
export function attachInput(canvas, gd, balance, view, actions) {
  canvas.addEventListener('pointerdown', (e) => {
    const { x, y } = toCanvas(canvas, e);       // toCanvas reuses no allocation beyond the return
    actions.onPress(x, y);
  });
}
```

Canvas has no "GameObjects" to pool, but the *same discipline* applies to any DOM you
create (score popups, particles): pre-create a fixed pool of elements, show/hide by
index, never `createElement` in the loop.

---

## 6. Game loop: one rAF, owns everything

```js
// main.js
import { balance, validate } from './balance.js';
import { allocateGameData } from './gameData.js';
import * as Logic from './logic.js';
import * as Board from './board.js';

validate(balance);                        // author-time gate, once
const gd = {};
allocateGameData(gd, balance);            // all TypedArrays allocated here, once
Logic.startRun(gd, balance);

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const view = makeView(canvas, balance);   // precomputed layout + colorCss strings
Board.attachInput(canvas, gd, balance, view, makeActions(gd, balance));

let last = performance.now();
function frame(now) {
  let dt = (now - last) / 1000; last = now;
  if (dt > 0.033) dt = 0.033;             // clamp so a tab-switch doesn't explode the sim
  Logic.tick(gd, balance, dt);            // all gameplay: pure, allocation-free
  Board.render(ctx, gd, balance, view);   // draw current state
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

The Board does **not** own its own loop. There is exactly one `requestAnimationFrame`
chain, in the Game/main module. Fixed-timestep substeps for stable physics go *inside*
`Logic.tick` (loop N times over `dt/N`), still allocation-free.

---

## 7. JavaScript per-frame allocation traps

These allocate on the heap and feed the GC. All are banned inside `tick`/`render` and
anything they call every frame. Each has an allocation-free rewrite.

| Trap (allocates every frame) | Allocation-free rewrite |
|---|---|
| `arr.map(...)`, `.filter(...)`, `.reduce(...)`, `.flatMap(...)` | explicit `for` loop writing into a pre-sized array |
| `[...a, ...b]`, `arr.concat(...)`, `Array.from(...)` | write into a pre-allocated array + count |
| `{ x, y }` object literals returned/created per item | write into parallel arrays, or reuse one scratch object |
| `const f = () => {...}` closures created per frame/item | hoist the function to module scope |
| destructuring that copies (`const {a,b} = obj` in a tight loop over objects) | index the arrays directly (`ax[i]`, `ay[i]`) |
| `` `score ${n}` `` / `str + n` built every frame | build the display string only when the value *changes* |
| `Math.hypot(dx, dy)` for a compare | compare squared: `dx*dx + dy*dy < r*r` (also skips sqrt) |
| `new Float32Array(...)` inside a logic function | allocate once in `allocateGameData`; `fill(0)` to clear |
| `JSON.parse/stringify` in the loop | keep state in TypedArrays; serialize only on save |
| array `push`/`splice` during play | array + count: `arr[count++]=v`; stack remove `v=arr[--count]` |

**Add / remove during play** (array + count):
```js
// add a ball
const i = gd.ballCount++;
gd.ballX[i] = x; gd.ballY[i] = y; gd.ballVX[i] = 0; gd.ballVY[i] = vy;

// remove ball i by swapping the last into its slot (O(1), order-independent)
const last = --gd.ballCount;
gd.ballX[i] = gd.ballX[last]; gd.ballY[i] = gd.ballY[last];
gd.ballVX[i] = gd.ballVX[last]; gd.ballVY[i] = gd.ballVY[last];
```

---

## 8. Testing DOD logic in Node (no browser, no deps)

Because Logic is pure functions over plain data, it tests trivially with Node's
built-in runner — no DOM, no framework. Only `balance.js`, `gameData.js`, `logic.js`
are imported; `board.js`/`main.js` (canvas/DOM) are verified manually.

```js
// test/logic.test.js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { balance } from '../src/balance.js';
import { allocateGameData } from '../src/gameData.js';
import { moveBalls } from '../src/logic.js';

test('gravity accelerates a ball downward', () => {
  const gd = {};
  allocateGameData(gd, balance);
  gd.ballCount = 1;
  gd.ballX[0] = 50; gd.ballY[0] = 10; gd.ballVX[0] = 0; gd.ballVY[0] = 0;
  moveBalls(gd, balance, 0.1);
  assert.ok(gd.ballVY[0] > 0);
  assert.ok(gd.ballY[0] > 10);
});
```

Run: `node --test`. TypedArrays work identically in Node and the browser, so tests
exercise the exact runtime data layout.

---

## 9. Determinism for tests

Randomness (board refill, spawns) must flow through a **seedable** generator so tests
are reproducible — never call `Math.random()` in Logic. A tiny seedable PRNG returned
as a closure, stored on `gameData`, injected once:

```js
// rng.js
export function mulberry32(seed) {
  let a = seed >>> 0;
  return function () {
    a = (a + 0x6d2b79f5) | 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
// allocateGameData: gd.rand = mulberry32(seed);
// logic uses gd.rand() — same seed => same sequence in tests and at runtime.
```

The generator closure is created once at allocation, not per frame — the one closure
that's fine because it never happens in the loop.

---

## 10. When (not) to reach for more

- **WebGL / instancing:** only when 2D canvas draw calls are the *measured* bottleneck.
  The data layout doesn't change — you're still iterating the same TypedArrays; you
  just upload them to the GPU. DOD makes this migration local.
- **Web Workers / SharedArrayBuffer:** TypedArrays backed by a `SharedArrayBuffer` can
  be handed to a worker for parallel simulation. Possible precisely *because* state is
  already in flat arrays — but add it only for a measured bottleneck, never by default.
- **WASM:** the same SoA data can be handed to a WASM module. Again: an incremental,
  local optimization enabled by the layout — not a starting requirement.

Plain TypedArray DOD in a single rAF loop is already allocation-free and
cache-friendly. Most browser games never need any of section 10.
