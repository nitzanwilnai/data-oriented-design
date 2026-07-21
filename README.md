# Data-Oriented Design — a language-agnostic architecture skill

A [Claude](https://claude.com/claude-code) **skill** that encodes the Data-Oriented Design (DOD)
architecture from *High Performance Unity Game Development (Using data-oriented design)* by
**Nitzan Wilnai** (Manning) — **generalized so it applies in any language**, with a first-class
**HTML5 / JavaScript / Canvas** track.

DOD is not a framework, an ECS, or a library. It's a coding paradigm built on three pillars —
**performance** (contiguous arrays for cache locality; data separated from logic), **reduced
complexity** (pure logic functions, clear data ownership), and **extensibility** (a feature's cost
stays constant — you only add data and logic, never refactor existing systems).

> **The DOD Mantra:** every logic function takes **data in**, **transforms** it, and writes **data
> out**. Logic functions never reach back into the presentation layer or the game loop.

## What's here

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | The skill itself: the five components (Balance / GameData / Logic / Board / Game), the data/logic/memory rules, Structure-of-Arrays by language, indices-instead-of-dictionaries, branchless thinking, the data-first feature flow, and an anti-pattern table. |
| [`references/patterns.md`](references/patterns.md) | Language-agnostic patterns: SoA, pure-logic anatomy, array-as-list/stack, index identity, pooling, indices-not-dictionaries. |
| [`references/anti-patterns.md`](references/anti-patterns.md) | OOP habits → DOD corrections, with generic + JS examples. |
| [`references/html-js.md`](references/html-js.md) | The HTML5/JS/Canvas track: TypedArrays, a Canvas Board, a `requestAnimationFrame` Game loop, and the per-frame-allocation traps unique to JS (closures, `.map/.filter`, spread, `Math.hypot` temporaries). |
| [`references/rust.md`](references/rust.md) | The native-Rust track: parallel `Vec`s vs `#[repr(C)]`/`Pod` GPU instance structs, a pure-logic crate + wgpu/winit native crate split, the read-only-UI/`UiAction` enforcement pattern, pre-allocated scratch buffers, and `cargo test`-based golden/parity testing — grounded in two shipped Rust DOD games. |

## The core idea

The book's five roles map onto any language:

```
Balance   — static config data; set at author/build time; never changes at runtime.
GameData  — mutable runtime state, held as parallel arrays (Structure-of-Arrays); only Logic modifies it.
Logic     — pure/free functions; the ONLY place GameData is modified.
Board     — presentation + input; reads GameData to render, writes input into GameData, calls Logic.
Game      — the top-level loop/orchestrator; owns Balance + GameData; drives Board.
```

The dependency arrow points one way: **Game → Board → Logic → (Balance, GameData)**. Identity is an
**index**, not an object. You **allocate up front** and **never allocate in the hot loop** (in managed
languages, a GC pause *is* the cache-miss stall).

For Unity specifically, prefer a dedicated `unity-dod-architecture` skill; use **this** one for
HTML/JS and every other language (C, C++, Rust, Go, Zig, C#, …).

## Install as a Claude skill

Drop the folder into your Claude skills directory so it's discoverable:

```bash
git clone https://github.com/nitzanwilnai/data-oriented-design.git \
  ~/.claude/skills/data-oriented-design
```

(or copy the files into `~/.claude/skills/data-oriented-design/`). Then it's available to Claude Code
whenever you're building or reviewing a game / real-time / simulation project in DOD style.

## The book

This skill distills the architecture from **[High Performance Unity Game Development](https://www.manning.com/) —
*Using data-oriented design*** by Nitzan Wilnai. The book covers the same ideas in depth, with the
original Unity/C# implementation.

## License

[MIT](LICENSE) © Nitzan Wilnai
