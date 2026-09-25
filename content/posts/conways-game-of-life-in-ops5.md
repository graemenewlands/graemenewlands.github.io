---
title: "Conway's Game of Life in OPS5: Declarative Cellular Automata in WebAssembly"
date: 2026-09-24T00:00:00-07:00
draft: false
featured_image: "/images/ops5-cover.jpeg"
description: "How we implemented John Conway's Game of Life as a declarative forward-chaining production rule system using the OPS5 Rete engine, compiled to WebAssembly, and running in the browser."
tags: ["Go", "WebAssembly", "OPS5", "Rete Algorithm", "Cellular Automata", "Rule Engines"]
---

Cellular automata like John Conway's **Game of Life** are almost universally taught and implemented imperatively: allocate a two-dimensional grid array, iterate over every row and column with nested `for` loops, count the 8 Moore neighbors, and write the next state to a backbuffer.

Following my recent work building [**`ops5` in Go**](https://github.com/graemenewlands/ops5) (and the deep dive in [*Building OPS5 in Go: Implementing the Rete Algorithm from Scratch*](/posts/building-ops5-in-go/)), I wanted to put the engine through its paces with something visually tangible, reactive, and non-trivial:

> **Can we implement Conway's Game of Life purely declaratively—with zero imperative loops—running live in the browser via WebAssembly?**

The result is now live on this site:
👉 **[Launch Conway's Game of Life (OPS5 WASM)](/life/)**  
⭐ Source code: [**github.com/graemenewlands/go-ops5-apps**](https://github.com/graemenewlands/go-ops5-apps)

Here is a breakdown of how the cellular automaton was modeled as production rules, how the Rete network executes each generation, and how it is wired into WebAssembly.

---

## 1. Modeling Cellular Automata as Facts (WMEs)

In OPS5, all knowledge and state is represented as **Working Memory Elements (WMEs)** adhering to schemas defined with `(literalize ...)`:

```lisp
(literalize step phase generation)
(literalize grid width height)
(literalize cell r c alive)
(literalize delta dr dc)
(literalize vote r c)
(literalize tally r c count)
(literalize next-cell r c alive)
```

### Sparse State Representation
A traditional grid array allocates $W \times H$ elements regardless of how many cells are actually alive. In our OPS5 model, working memory stores **only live cells**:

```lisp
(cell ^r 12 ^c 15 ^alive 1)
```

Dead cells do not exist as facts in working memory. This means computation is proportional to active cell activity rather than total grid surface area.

### Pre-Asserted Delta Vectors
At startup, the Go wrapper asserts the 8 directional delta vectors representing the Moore neighborhood:

```go
deltas := [][2]int64{
    {-1, -1}, {-1, 0}, {-1, 1},
    { 0, -1},          { 0, 1},
    { 1, -1}, { 1, 0}, { 1, 1},
}
```

These static facts sit permanently in working memory, ready to join against dynamic cell facts in the Rete beta network.

---

## 2. The Five-Stage Declarative Pipeline

Rather than computing states sequentially, each generation flows through five declarative stages coordinated by a single control fact: `(step ^phase <phase> ^generation <g>)`.

```
+------------+       +-------------+       +------------------+
| Stage 1    |  -->  | Stage 2     |  -->  | Stage 3          |
| Emit Votes |       | Tally Votes |       | Evaluate Conway  |
+------------+       +-------------+       +------------------+
                                                    |
+------------+       +-------------+                |
| Stage 5    |  <--  | Stage 4     |  <-------------+
| Quiescence |       | Clear &     |
| (Done)     |       | Promote     |
+------------+       +-------------+
```

### Stage 1: Emitting Neighbor Votes

For every live cell, the engine joins `cell` with `delta` and the `grid` dimensions to emit a `vote` fact for each neighboring coordinate. Toroidal boundary wrapping is evaluated via modulo arithmetic directly on the RHS:

```lisp
(p emit-vote [salience 10]
    (step ^phase emit)
    (grid ^width <w> ^height <h>)
    (cell ^r <r> ^c <c> ^alive 1)
    (delta ^dr <dr> ^dc <dc>)
  -->
    (bind <nr> (compute <r> + <dr> + <h> % <h>))
    (bind <nc> (compute <c> + <dc> + <w> % <w>))
    (make vote ^r <nr> ^c <nc>)
)

;; Once all live cell/delta joins are exhausted:
(p transition-emit-to-tally [salience 1]
    <st> (step ^phase emit)
  -->
    (modify <st> ^phase tally)
)
```

Due to Rete's refraction guarantee, `emit-vote` fires exactly once per `(cell, delta)` pair. When no more activations remain, the low-salience transition rule fires, advancing the phase to `tally`.

---

### Stage 2: Tallying Votes (The Consumption Accumulator Pattern)

Multiple live cells may vote for the same target coordinate. We aggregate these votes using a classic OPS5 **consumption accumulator pattern**:

```lisp
;; Initialize a new coordinate tally
(p init-tally [salience 20]
    (step ^phase tally)
    <v> (vote ^r <r> ^c <c>)
    -(tally ^r <r> ^c <c>)
  -->
    (make tally ^r <r> ^c <c> ^count 1)
    (remove <v>)
)

;; Increment an existing coordinate tally
(p inc-tally [salience 20]
    (step ^phase tally)
    <t> (tally ^r <r> ^c <c> ^count <cnt>)
    <v> (vote ^r <r> ^c <c>)
  -->
    (bind <next> (compute <cnt> + 1))
    (modify <t> ^count <next>)
    (remove <v>)
)
```

Each firing consumes a `vote` fact with `(remove <v>)`. When working memory has zero remaining votes (`-(vote)`), the engine transitions to `evaluate`:

```lisp
(p transition-tally-to-eval [salience 1]
    <st> (step ^phase tally)
    -(vote)
  -->
    (modify <st> ^phase evaluate)
)
```

---

### Stage 3: Evaluating Conway's Rules

With neighbor tallies populated, Conway's standard $B3/S23$ rules are expressed using pattern tests and negative conditions:

1. **Birth ($B3$)**: A dead cell with exactly 3 live neighbors becomes alive:
   ```lisp
   (p birth [salience 30]
       (step ^phase evaluate)
       <t> (tally ^r <r> ^c <c> ^count 3)
       -(cell ^r <r> ^c <c> ^alive 1)
     -->
       (make next-cell ^r <r> ^c <c> ^alive 1)
       (remove <t>)
   )
   ```

2. **Survival ($S23$)**: A currently living cell with 2 or 3 neighbors survives:
   ```lisp
   (p survival-2 [salience 30]
       (step ^phase evaluate)
       <t> (tally ^r <r> ^c <c> ^count 2)
       (cell ^r <r> ^c <c> ^alive 1)
     -->
       (make next-cell ^r <r> ^c <c> ^alive 1)
       (remove <t>)
   )

   (p survival-3 [salience 30]
       (step ^phase evaluate)
       <t> (tally ^r <r> ^c <c> ^count 3)
       (cell ^r <r> ^c <c> ^alive 1)
     -->
       (make next-cell ^r <r> ^c <c> ^alive 1)
       (remove <t>)
   )
   ```

3. **Death by Under/Overpopulation**: Any remaining tallies ($< 2$ or $> 3$, or dead cells with 2 neighbors) simply get discarded:
   ```lisp
   (p discard-tally [salience 20]
       (step ^phase evaluate)
       <t> (tally ^r <r> ^c <c>)
     -->
       (remove <t>)
   )
   ```

---

### Stages 4 & 5: Promoting and Quiescence

Once all tallies are processed:
1. `clear-cells`: Retracts the previous generation's `cell` facts.
2. `promote`: Promotes all `next-cell` facts into `cell` facts.
3. `quiescence`: Increments the generation counter and sets the phase back to `ready`:

```lisp
(p finish-step [salience 1]
    <st> (step ^phase promote ^generation <g>)
    -(next-cell)
  -->
    (bind <next-g> (compute <g> + 1))
    (modify <st> ^phase ready ^generation <next-g>)
)
```

With no rules matched, the Rete conflict set becomes empty. The engine naturally enters **quiescence** and returns control to Go.

---

## 3. WebAssembly Integration

The simulation engine is compiled directly to WebAssembly (`GOOS=js GOARCH=wasm`) using Go 1.23. The Go host exposes a synchronous bridge via `syscall/js`:

| JS Function | Purpose |
| :--- | :--- |
| `window.lifeInit(width, height, [ruleSource])` | Instantiates engine, parses rules, asserts grid & deltas. |
| `window.lifeStep()` | Modifies `step` phase to `emit`, calls `eng.Run(0)`, and returns step statistics. |
| `window.lifeToggleCell(r, c)` | Toggles individual working memory facts. |
| `window.lifeLoadPattern(name, r, c)` | Loads presets (Glider, Gosper Gun, Pulsar, Acorn, etc.). |
| `window.lifeGetRuleSource()` | Exports the active OPS5 rule source code. |

In the browser, an HTML5 `<canvas>` rendering loop drives the simulation via `requestAnimationFrame`:

```javascript
async function stepSimulation() {
  const stats = window.lifeStep();
  if (stats.success) {
    updateCanvas(stats.cells);
    updateMetrics(stats);
  }
}
```

---

## 4. Live Rule Hot-Reloading

Because the OPS5 rule parser and Rete network compiler are bundled directly inside the WebAssembly binary, the application features an in-browser **Rule Inspector & Editor**.

You can click **OPS5 Rules** in the top navigation of the app, modify the rule definitions on the fly, and click **Apply & Hot-Reload Rules**:

- Want to test **HighLife** ($B36/S23$)? Add a rule for `(count 6)` births!
- Want to test **Seeds** ($B2/S$)? Remove the survival rules and change birth count to 2.
- The Wasm engine re-compiles the Rete discrimination tree instantly in memory without reloading the page.

---

## Try It Out

- 🎮 **Live App**: [https://graemenewlands.com/life/](/life/)
- 💻 **App Repository**: [github.com/graemenewlands/go-ops5-apps](https://github.com/graemenewlands/go-ops5-apps)
- ⚙️ **Core OPS5 Engine**: [github.com/graemenewlands/ops5](https://github.com/graemenewlands/ops5)
