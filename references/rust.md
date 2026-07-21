# DOD in Rust

The concrete translation of the DOD architecture to a native Rust game — a workspace
with a pure-logic crate and a wgpu/winit crate on top. This track is grounded in two
real, shipped Rust DOD games: **Luminko** (a Steam pachinko-roguelite; `core` +
`native` crates) and **TrainGame / Keypress Express** (a Steam desktop-overlay train
idle game; `keypress-core` + `keypress-native`). Both independently converged on the
same shape — a rendering-free logic crate consumed by a native crate — without an ECS
crate in sight.

Two ideas make DOD *stricter* in Rust than in a garbage-collected language:

1. **`Vec<T>` per field is already a contiguous, unboxed buffer** — no boxing, no GC.
   But there's no collector to hide a mistake: every `Vec::new()` / `.clone()` /
   `.collect()` inside a hot loop is a real `malloc`. "Allocate once, never in the
   loop" isn't about dodging a GC pause — there's no GC to trigger.
2. **The borrow checker enforces the discipline other tracks get by convention.**
   `fn tick(gd: &mut GameData, balance: &Balance, dt: f64)` isn't just a naming
   convention — the compiler won't let two things mutate `gd` at once, and a
   read-only UI function that takes `&GameData` is *physically incapable* of
   mutating gameplay state. DOD's rules and Rust's ownership rules point the same way.

---

## 1. Structure of Arrays with parallel `Vec`s

### Array of structs (avoid)
```rust
struct Ball { x: f32, y: f32, vx: f32, vy: f32 }
let mut balls: Vec<Ball> = Vec::new();   // one heap object per ball; fields interleaved
```

### Structure of arrays (use)
```rust
// core/src/gamedata.rs (Luminko) — one Vec per field, allocated once, sized from Balance.
pub struct GameData {
    pub ball_x: Vec<f32>, pub ball_y: Vec<f32>,
    pub ball_vx: Vec<f32>, pub ball_vy: Vec<f32>,
    pub ball_count: i32,   // array + count, not a growable list
}
// logic/ball.rs — identity is the index i; no per-ball object exists.
pub fn spawn_ball(gd: &mut GameData, x: f64, vy: f64) -> i32 {
    let i = gd.ball_count as usize;
    gd.ball_count += 1;
    gd.ball_x[i] = x as f32; gd.ball_vy[i] = vy as f32;
    i as i32
}
```
TrainGame's `GameData` is the same shape for a different domain — cars in a train
yard — with the rule spelled out in the doc comment:
```rust
// Owned cars — SoA parallel arrays; index = car slot.
// All arrays must stay in lockstep (same length at all times).
pub car_base_type_id: Vec<CarTypeId>,
pub car_income_pct: Vec<f64>,
```
Divergence: Luminko indexes with raw `i32` enum constants (`BALL_STANDARD: i32 =
0`); TrainGame wraps ids in newtypes (`CarTypeId`, `DestId`) for compile-time
protection against mixing up which kind of id you're holding. Both are still
index-based identity.

**Choose the field width deliberately**: `f32` for positions/velocities (half the
memory of `f64`, denser cache lines); `i32`/`u8` for counts, levels, enum tags.
Reserve `#[repr(C)]` + `derive(Pod, Zeroable)` (`bytemuck`) for **GPU instance
structs**, never entity state — Luminko's `GameData` fields are plain `Vec<f32>`;
its renderer's per-instance struct is `#[repr(C)] #[derive(Pod, Zeroable)] struct
GlowInst { center: [f32; 2], radius: f32 }`, bitcast straight into a GPU buffer via
`bytemuck::cast_slice`. `GameData` never crosses that boundary, so it never needs
the annotation.

**A 2D grid** is a flat `Vec`, indexed `row * width + col`:
```rust
pub grid_level: Vec<i32>,                        // 0 = empty, else 1..max_level
let idx = (r * balance.grid_w + c) as usize;      // gd.grid_level[idx]
```

---

## 2. The crate layout

A pure-logic crate with **zero** rendering dependencies, plus a native crate that
depends on it and owns wgpu/winit/egui — a real, checkable boundary (`grep -rn
"wgpu\|winit" core/src/**/*.rs` returns nothing in either game).

```
Cargo.toml            [workspace] members = ["core", "native"]     (Luminko)
core/Cargo.toml        deps: serde, serde_json                     — no wgpu, no winit
core/src/
  balance.rs           config + author-time validate()
  gamedata.rs          GameData: parallel Vecs + counts + phase/enum fields
  rng.rs               seeded PRNG (no rand::thread_rng, no Instant::now)
  logic/game.rs, ball.rs, grid.rs, tile.rs    — free fns, split by domain
native/Cargo.toml      deps: wgpu, winit, egui, matchinko-core (path dep)
native/src/
  main.rs              the ApplicationHandler: owns the loop, applies UiAction
  fairy.rs, atlas.rs   wgpu instanced renderers (pre-allocated scratch buffers)
  ui.rs                egui panels; reads GameData, returns UiAction — never mutates it
```
Logic functions are `pub fn name(gd: &mut GameData, balance: &Balance, ...)` — free
functions, never methods on `GameData`. `native` is the only crate touching
`wgpu`/`winit`/`egui`.

TrainGame confirms this is a general pattern, not a one-off — the same split
(`keypress-core`: serde only; `keypress-native`: wgpu/winit/egui/steamworks), even
without a formal Cargo `[workspace]` (each crate keeps an independent `Cargo.toml`).
The boundary held anyway: it's a dependency-choice discipline, not a tooling
requirement — proven further by TrainGame shipping a *second*, Tauri+JS/HTML front
end that consumes the same rendering-agnostic core.

---

## 3. Balance + author-time validation

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Balance {
    pub grid_w: i32, pub grid_h: i32, pub gravity: f64,
    pub score_by_level: Vec<i32>,   // author-time table; index by effective level
}
impl Default for Balance {
    fn default() -> Balance { Balance { grid_w: 8, grid_h: 6, /* ... */ } }
}
/// Every invariant violation (empty = valid), never a panic — callers choose how to
/// surface it. Runtime gameplay code never re-checks any of this.
pub fn validate(b: &Balance) -> Vec<String> {
    let mut errs = Vec::new();
    if b.num_colors as usize > b.color_css.len() { errs.push("numColors > colorCss.length".into()); }
    errs
}
#[test]
fn balance_has_required_fields_and_valid_geometry() {
    assert_eq!(validate(&Balance::default()), Vec::<String>::new());
}
```
`validate` runs in a test (author/CI time), not on every load. Once it passes, every
`logic/` function reads `balance.whatever` with zero defensive checks — the type
system guarantees the field exists; `validate` guarantees the value makes sense.

---

## 4. Logic: pure, allocation-free, index-based

```rust
// logic/ball.rs — a free fn over &mut GameData; no IO, no rendering, no wall clock.
pub fn step_balls(gd: &mut GameData, balance: &Balance, dt: f64) -> i32 {
    let mut i: usize = 0;
    while i < gd.ball_count as usize {
        gd.ball_vy[i] += (balance.gravity * dt) as f32;
        gd.ball_x[i] += gd.ball_vx[i] * dt as f32;
        collide_pegs(gd, balance, i);           // may swap-remove — see below
        if gd.ball_y[i] as f64 > balance.canvas_h { /* drain: swap-remove */ } else { i += 1; }
    }
    0
}
```
TrainGame's `tick` is the same shape for an idle-game domain — no physics, still a
plain free function, no allocation, no IO: `if gd.paused { return; } economy::
refill_stations(gd, b, now_ms); if gd.trip_active { trip::travel_step(gd, b, dt); }`.

**Array + count**, not a growable list, for add/remove during play — O(1) swap-remove
moves every parallel field of the last live entity into the freed slot:
```rust
gd.ball_count -= 1;
let last = gd.ball_count as usize;
gd.ball_x[i] = gd.ball_x[last]; gd.ball_y[i] = gd.ball_y[last];
gd.ball_vx[i] = gd.ball_vx[last]; gd.ball_vy[i] = gd.ball_vy[last];
```

**Scratch buffers live on `GameData`**, allocated once, never inside the function
that uses them — a flood-fill visited mask, a DFS stack, a particle pool:
```rust
pub scratch_seen: Vec<i32>,   // flood-fill visited mask, len = gridW*gridH — cleared, never reallocated
pub scratch_stack: Vec<i32>,  // DFS stack, reused every match-check
```

**Prefer squared distance** — skip the `sqrt` until the real distance is needed:
`if dx*dx + dy*dy < min*min { /* collision */ }`.

**The borrow checker helps SoA, it doesn't fight it.** Every field is a separate
`Vec` directly on `GameData`, so `&mut GameData` gives checked, non-aliasing access
to all of them at once — no `RefCell`, no runtime borrow panics, unlike ECS crates
that lean on interior mutability for flexible per-component borrowing. The price —
two logic functions can't both hold `&mut GameData` at once — is exactly the DOD
rule "one function owns this step's mutation," now compiler-enforced.

---

## 5. Board: wgpu/winit render + input

The Board is the only code touching `wgpu`/`winit`/`egui`. It reads `GameData` to
draw, and its UI function's *type signature* proves it can't sneak in a mutation.

**Pre-allocated instance buffers + scratch `Vec`s**, cleared and refilled every
frame, never reallocated. Luminko's `FairyRenderer` pairs each GPU buffer with a
scratch `Vec` of the matching instance type, both sized once in `new()`:
```rust
pub struct FairyRenderer { glow_buf: wgpu::Buffer, glow_scratch: Vec<GlowInst> } // one pair per kind
impl FairyRenderer {
    pub fn new(device: &wgpu::Device) -> Self {
        let glow_buf = device.create_buffer(&wgpu::BufferDescriptor {
            size: (GLOW_CAP * std::mem::size_of::<GlowInst>()) as u64, /* ... */
        });
        FairyRenderer { glow_buf, glow_scratch: Vec::with_capacity(GLOW_CAP) }
    }
    pub fn draw(&mut self, queue: &wgpu::Queue, gd: &GameData) {
        self.glow_scratch.clear();                             // capacity retained, no realloc
        for i in 0..gd.ball_count as usize { /* push into scratch Vecs */ }
        if !self.glow_scratch.is_empty() {
            queue.write_buffer(&self.glow_buf, 0, bytemuck::cast_slice(&self.glow_scratch));
        }
    }
}
```
TrainGame's `SceneData` is a stricter variant: a manual `count` instead of `Vec::
clear()`/`push()`, and it **panics** on overflow instead of silently reallocating —
`assert!(sd.count < sd.instances.len(), "buffer full"); sd.instances[sd.count] =
inst; sd.count += 1;` — turning a wrong capacity estimate into a test failure
instead of a stall discovered in production. (Luminko's `atlas.rs` has a looser
third variant for screens that don't run every tick: a grow-only buffer, resized
only when demand exceeds capacity.)

**The read-only-UI / `UiAction` pattern is the enforcement trick** for "Board never
mutates GameData except by calling Logic":
```rust
pub enum UiAction { StartRun, ResolveEvent(i32), SelectShopItem { kind: i32, slot: i32 } }
pub fn run_ui(ctx: &egui::Context, gd: &GameData, balance: &Balance) -> Option<UiAction> {
    /* reads gd, builds panels, returns what the player clicked — cannot write to gd */
}
```
```rust
// main.rs — applied only after the &GameData borrow above has ended.
// "this is the ONLY place `main.rs` mutates `gd` in response to a `UiAction`"
match ui_action {
    Some(ui::UiAction::StartRun) => start_run(&mut self.gd, &self.balance),
    Some(ui::UiAction::ResolveEvent(opt)) => resolve_event(&mut self.gd, &self.balance, opt),
    None => {}
}
```
Because `run_ui` takes `&GameData`, not `&mut GameData`, the compiler rejects any
attempt by UI code to mutate gameplay state — the rule isn't just documented, it's
the only code the type checker will accept.

---

## 6. Game loop: one loop, owns everything

```rust
impl ApplicationHandler for App {
    fn about_to_wait(&mut self, _el: &ActiveEventLoop) {
        self.advance();
        if let Some(w) = &self.window { w.request_redraw(); }
    }
}
impl App {
    fn advance(&mut self) {
        let mut dt = (Instant::now() - self.last_tick).as_secs_f64();
        if dt > MAX_DT { dt = MAX_DT; }        // clamp so a stall doesn't explode the sim
        tick(&mut self.gd, &self.balance, dt); // ALL gameplay, allocation-free
    }
    fn render(&mut self) { /* draws current self.gd; never advances gameplay */ }
}
```
One `ApplicationHandler`, one place `dt` is computed and clamped, one call into
`tick`. `render` only reads; `advance` is the only place logic runs. Fixed-timestep
substeps for stable physics belong *inside* `tick` (loop N times over `dt/N`), still
allocation-free.

TrainGame's loop is event-driven rather than continuously redrawing — `ControlFlow::
Wait`, redrawing only on a dirty flag, since it's a desktop overlay meant to idle
near 0% CPU rather than a full-screen 60Hz game. Same order (input, tick, render);
it just skips render when nothing changed.

---

## 7. Rust per-frame allocation traps

All of these call the allocator inside `tick`/`draw`. `cargo clippy` catches some
(`needless_collect`) but not most — the discipline is manual.

| Trap (allocates every frame) | Allocation-free rewrite |
|---|---|
| `Vec::new()` / `vec![...]` inside `tick`/`draw` | allocate once (`GameData::new`, `Renderer::new`); reuse via `.clear()` |
| `.iter().map(...).collect::<Vec<_>>()` | explicit `for` loop writing into a pre-sized scratch `Vec` |
| `Box<dyn Trait>` dispatch in the hot loop | a type-id enum indexing plain data, or a `match` over it |
| `format!(...)` / `String` built every frame | build the display string only when the value changes |
| gratuitous `.clone()` of a `Vec`/struct per item | index the arrays directly (`gd.ball_x[i]`), or pass `&` |
| `HashMap`/`BTreeMap` lookups | integer id indexing a `Vec`; a map only as a one-time load-time remap |
| `Vec::push`/`remove(k)` during play | array + count: `arr[count] = v; count += 1;`; swap-remove |
| a `Vec` `.push()`ed without `.clear()`ing first | unbounded growth — always `clear()` before refilling a per-frame scratch buffer |
| `dx.hypot(dy)` for a compare | compare squared distance; skip the `sqrt` |

TrainGame's `assert!`-guarded `push` (section 5) is worth adopting broadly: a
capacity-exceeded bug then surfaces as a failing test during development, not a
silent reallocation discovered as a stall in production.

---

## 8. Testing DOD logic with `cargo test` (no GPU)

Because the logic crate has zero rendering dependencies, `cargo test --lib` in
`core` runs fully headless — no window, no adapter, no device. Luminko's `core`
carries 347 `#[test]` functions; TrainGame's runs 94 passing — none creating a
`wgpu::Instance`.

**Byte-parity / golden tests** are the strongest check when a crate is a *port* of
an existing reference (Luminko ported its game from a JS/Canvas prototype — the JS
is the source of truth, and the golden values below were captured once from a live
`node` run with the same seed and scripted input sequence):
```rust
struct Golden { seed: u64, frames: i32, score: f64 }

fn run_and_assert(g: &Golden) {
    let mut gd = GameData::new_with_seed(&Balance::default(), g.seed);
    start_run(&mut gd, &b);
    while gd.phase == PHASE_PACHINKO && f < 12000 { tick(&mut gd, &b, 1.0 / 60.0); f += 1; }
    assert_eq!(gd.score, g.score, "seed {}: score diverged from the JS golden", g.seed);
}
#[test]
fn full_run_matches_js_seed_2026() { run_and_assert(&Golden { seed: 2026, frames: 293, score: 230.0 }); }
```
Any future refactor that shifts gameplay math trips this test immediately.

When there's no external reference to port from, an equally valid headless test
asserts on **invariants** instead of literal golden values. TrainGame's `core/src/
sim.rs` runs synthetic "optimal"/"casual" player policies through the real `logic::
{trip, shop, economy}` functions for hundreds of simulated days and asserts tuning
targets (`sim_upgrades_spread_across_towns`) — useful when a game is designed in
Rust from scratch and there's no golden trace to compare against, only intent.

---

## 9. Determinism for tests

Randomness in Logic must flow through a **seeded** generator, never `rand::
thread_rng()` or `Instant::now()` read internally — pass `dt`/a seed in.

```rust
// core/src/rng.rs — a seeded PRNG (mulberry32)
pub struct Rng { a: u32 }
impl Rng {
    pub fn new(seed: u64) -> Self { Rng { a: seed as u32 } }
    pub fn next_f64(&mut self) -> f64 {
        self.a = self.a.wrapping_add(0x6d2b79f5);
        let mut t = (self.a ^ (self.a >> 15)).wrapping_mul(1 | self.a);
        t = t.wrapping_add((t ^ (t >> 7)).wrapping_mul(61 | t)) ^ t;
        ((t ^ (t >> 14)) as f64) / 4294967296.0
    }
}
#[test]
fn deterministic_per_seed() {
    let (mut a, mut b) = (Rng::new(42), Rng::new(42));
    assert_eq!(a.next_f64(), b.next_f64());
}
```
Luminko goes further: `GameData` holds **two** `Rng`s — `rand` for the gameplay
stream (fill, refill, scoring rolls) and a separate `anim_rand` for cosmetic
randomness (particle angles, floater wobble), call sites commented "visual RNG —
never the gameplay stream." Any draw on the gameplay path shifts the sequence for
everything after it, so turning particle effects on/off must never perturb a replay.

TrainGame reaches determinism differently, just as validly: no stored `Rng` struct,
no `rand` dependency at all — a hand-rolled FNV-1a hash plus an LCG, reseeded per
call from a caller-supplied `salt`/`now_ms` parameter instead of a `GameData` field.
Both satisfy the same rule — a seed lives somewhere reachable and deterministic, and
logic never reads the wall clock for a gameplay outcome — whether it sits in a
struct or is threaded through parameters is a choice, not a requirement.

---

## 10. When (not) to reach for an ECS crate

Neither Luminko nor TrainGame — two real, shipped-on-Steam Rust games — uses
`bevy`, `hecs`, `legion`, or any other ECS crate. Parallel `Vec<T>` fields indexed
by a `usize`/`i32` slot already *are* data-oriented design: contiguous,
cache-friendly, allocation-free once sized, and (section 4) the borrow checker
enforces non-aliasing access for free — many ECS crates reintroduce *runtime*
borrow checking (`RefCell`/`UnsafeCell` component storage) specifically to recover
the flexibility a plain struct's compile-time borrows already give you here.

An ECS's real value is **dynamic composition at scale**: entities whose component
set varies and changes at runtime, queried by arbitrary conjunctions of component
types, numbering in the thousands across many archetypes. Neither game needs that —
Luminko has a fixed, fully-known handful of entity kinds (balls, particles,
floaters, rings, movers), each with a fixed field set decided at design time;
TrainGame's cars/engines/stations are the same shape. A hand-written `GameData`
struct is simpler here: no archetype churn, no query-planning overhead, and the
data-first feature flow ("what data, what logic, what tick call") says exactly
where a new field goes without learning a framework's own conventions.

Reach for an ECS crate only when both hold: (a) entity composition is genuinely
dynamic and heterogeneous — many archetypes, added/removed at runtime — and (b) a
plain struct's growing field list is the actual bottleneck to extend, not merely
large. Until then, an ECS is ceremony over `Vec`s you already have.
