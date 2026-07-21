# DOD Anti-Patterns Reference (language-agnostic)

Common OOP habits that violate DOD, with corrections. Examples lean on JS since this
skill's primary track is HTML/JS, but the corrections are language-neutral.

## Contents
1. Data and Logic Coupled in Objects
2. Array of Objects Instead of Parallel Arrays
3. Growable Lists in Hot Paths
4. Per-Frame Allocation (the GC trap)
5. Logic Reaching Into Presentation / Input
6. Inheritance for Entity Types
7. Over-Abstraction of Data Operations
8. Defensive Runtime Checks on Config
9. Dictionaries in the Hot Path
10. Early Returns in Tick Functions
11. Per-Item Liveness Checks

---

## 1. Data and Logic Coupled in Objects

### Anti-pattern
```js
class Ball {
  constructor(x, y) { this.x = x; this.y = y; this.vx = 0; this.vy = 0; }
  update(dt, g) { this.vy += g * dt; this.x += this.vx * dt; this.y += this.vy * dt; }
  isGone(bottom) { return this.y > bottom; }
}
```

### Correction
```js
// data (parallel arrays, allocated once)
gd.ballX = new Float32Array(n); gd.ballY = new Float32Array(n);
gd.ballVX = new Float32Array(n); gd.ballVY = new Float32Array(n);

// logic (pure functions, no `this`)
export function moveBalls(gd, balance, dt) {
  for (let i = 0; i < gd.ballCount; i++) {
    gd.ballVY[i] += balance.gravity * dt;
    gd.ballX[i]  += gd.ballVX[i] * dt;
    gd.ballY[i]  += gd.ballVY[i] * dt;
  }
}
export function ballIsGone(gd, i, bottom) { return gd.ballY[i] > bottom; }
```

---

## 2. Array of Objects Instead of Parallel Arrays

### Anti-pattern
```js
const enemies = [];
enemies.push({ x, y, vx, vy, hp });        // scattered heap objects, boxed numbers
```

### Correction
```js
gd.posX = new Float32Array(n); gd.posY = new Float32Array(n);
gd.velX = new Float32Array(n); gd.velY = new Float32Array(n);
gd.hp   = new Int32Array(n);                // Structure of Arrays; identity = index
```

---

## 3. Growable Lists in Hot Paths

### Anti-pattern
```js
gd.alive.push(i);        // may reallocate the backing store
gd.alive.splice(k, 1);   // shifts + churns; no control over allocation
```

### Correction
```js
// array + count
gd.aliveIdx[gd.aliveCount++] = i;         // add
gd.aliveIdx[k] = gd.aliveIdx[--gd.aliveCount]; // swap-remove, O(1)
```

---

## 4. Per-Frame Allocation (the GC trap)

In a garbage-collected language you cannot free — you can only avoid allocating. Every
allocation in the loop feeds the collector, and a collection is a dropped frame.

### Anti-pattern
```js
function tick(gd, balance, dt) {
  const alive = gd.balls.filter(b => !b.gone);        // new array every frame
  const positions = alive.map(b => ({ x: b.x, y: b.y })); // more garbage
  const label = `Score: ${gd.score}`;                 // new string every frame
}
```

### Correction
```js
function tick(gd, balance, dt) {
  for (let i = 0; i < gd.ballCount; i++) { /* update in place, no allocation */ }
}
// build the score string only when the value changes, in the board — not every frame.
```

---

## 5. Logic Reaching Into Presentation / Input

### Anti-pattern
```js
function tick(gd, balance, dt) {
  const ctx = canvas.getContext('2d');   // logic touching the canvas
  if (mouse.down) gd.launchX = mouse.x;  // logic reading the input device
  ctx.fillRect(...);                     // logic drawing
}
```

### Correction
```js
// board.js — input handler stages intent into gameData; board renders after logic
canvas.addEventListener('pointermove', e => { gd.aimX = toCanvasX(canvas, e); });

// logic.js — touches only plain data
export function tick(gd, balance, dt) { /* uses gd.aimX; never the DOM */ }
```
Logic never references the canvas, DOM, events, or timers. The Board owns all of that.

---

## 6. Inheritance for Entity Types

### Anti-pattern
```js
class Enemy {}
class FastEnemy extends Enemy {}
class TankEnemy extends Enemy {}   // polymorphic array: cache misses + vtable dispatch
```

### Correction
```js
// one type-id per slot; config holds per-type data indexed by that id
gd.typeId = new Int32Array(n);
const v = balance.typeVel[gd.typeId[i]];   // all entities share the same arrays
```

---

## 7. Over-Abstraction of Data Operations

### Anti-pattern
```js
class FastList {
  add(v) { this.a[this.n++] = v; }
  removeAt(k) { this.a[k] = this.a[--this.n]; }
}
list.add(i);   // indirection hiding one line of array manipulation
```

### Correction
```js
gd.aliveIdx[gd.aliveCount++] = i;              // explicit; obvious; no indirection
gd.aliveIdx[k] = gd.aliveIdx[--gd.aliveCount];
```
Abstractions hide what the code does with data. Tomorrow's case may differ — don't
prematurely generalize a one-liner.

---

## 8. Defensive Runtime Checks on Config

### Anti-pattern
```js
export function allocate(gd, balance) {
  if (balance.maxBalls > 0 && balance.maxBalls < 100000)  // readers now wonder why
    gd.ballX = new Float32Array(balance.maxBalls);
}
```

### Correction
```js
// author/build time:
export function validate(b) { if (b.maxBalls <= 0) throw new Error('maxBalls > 0'); }

// runtime trusts the data:
export function allocate(gd, balance) { gd.ballX = new Float32Array(balance.maxBalls); }
```

---

## 9. Dictionaries in the Hot Path

### Anti-pattern
```js
const speed = enemyTypes.get('zombie').velocity;   // hash + pointer chase every frame
```

### Correction
```js
const speed = balance.typeVel[gd.typeId[i]];        // integer index into a TypedArray
```
Allow a map only as a one-time load-time remap (saved id/name → current index).

---

## 10. Early Returns in Tick Functions

### Anti-pattern
```js
function boardTick(gd, balance, dt) {
  const over = Logic.tick(gd, balance, dt);
  if (over) { showGameOver(); return; }   // skips the visual sync below
  syncVisuals(gd, balance);               // player never sees the final frame
}
```

### Correction
```js
function boardTick(gd, balance, dt) {
  const over = Logic.tick(gd, balance, dt);
  syncVisuals(gd, balance);   // always sync — player sees final state
  if (over) showGameOver();   // act on results at the end
}
```
Game-over is rare; there's no perf reason to skip the sync, and early returns risk
skipping cleanup/visual work.

---

## 11. Per-Item Liveness Checks

### Anti-pattern
```js
for (let i = 0; i < gd.count; i++) {
  if (!gd.alive[i]) continue;   // a branch every iteration
  update(gd, i);
}
```

### Correction
```js
// keep a separate live index list; the branch disappears
for (let k = 0; k < gd.liveCount; k++) { const i = gd.liveIdx[k]; update(gd, i); }
```
