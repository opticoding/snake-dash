# SNAKE DASH

**An ambient snake bot that plays itself — forever.**

A mathematical art project in at attempt to maximize both shortest path, randomness and score at the same time. It will try to beat its own score, which will statistically always be possible but will take forever.

Try it:
https://opticoding.github.io/snake-dash/

<img width="985" height="667" alt="image" src="https://github.com/user-attachments/assets/eb00c934-bd25-4b7d-97bf-77041f150688" />

---

## What is this?

Snake Dash is a single HTML file you can open in any browser and leave running. There is no player, no goal to complete. The algorithm pilots the snake across the grid continuously, eating, growing, and starting fresh when a round ends — all on its own, creating unique and interesting patterns, and strange untangling.

It was built as a **living screensaver**: something beautiful to leave on a second monitor, a digital photo frame, a lobby display, or a TV. Designed to look good for any length of time.

---

## Every round is a first

Here is a thought worth sitting with.

The snake's path each round is determined by where food randomly appears. The grid on a typical screen holds somewhere between 500 and 2,000 cells. Each piece of food spawns at any free cell. The sequence of food placements across a full round — combined with the bot's reactions — produces a pattern of movement that is, in any practical sense, **unique in the history of the universe**.

Consider the smallest grid this runs on — roughly **7×10 cells, just 70 squares**. The number of possible food sequences on that tiny board is approximately **10⁹⁸** — nearly a googol (10¹⁰⁰). That is already larger than the estimated number of atoms in the observable universe (10⁸⁰), on a grid you could draw on a napkin.

The default grid is closer to 40×25 — around 1,000 cells. The number of possible sequences there is approximately **10²⁵⁰⁰**. That number has no useful analogy. It dwarfs atoms, dwarfs time, dwarfs any physical quantity ever measured or imagined. Even if every atom in the universe had been running this simulation since the Big Bang, the fraction of possible rounds explored would round to zero.

What you are watching is not a loop. It is not a preset animation. It is a genuinely novel path being drawn for the first time, and the last.

---

## Use cases

- **Screensaver** — open `index.html` in a browser, set fullscreen, walk away
- **Digital frame / ambient display** — plug into any screen with a browser
- **Dynamic background art** — a living piece that never repeats
- **Desk decoration** — meditative, non-distracting motion in the corner of your vision
- **Conversation piece** — "what is that?" is a common reaction

No installation. No server. No dependencies. One file.

---

## Controls

### Speed slider
Located at the top of the canvas. Drag left for slower, right for faster.

### AI mode toggle
Cycles through four intelligence levels on each click.

| Label | Colour | What it does |
|-------|--------|--------------|
| **Normal** | Default | Standard AI — Dijkstra + tail-chase fallback, no simulation |
| **Smart** | Dark green | Adds a *targeted* lookahead veto: when the snake hits its own body or wall and must choose left or right, simulates `LOOKAHEAD_STEPS_B` (100) ticks ahead for each option. Only overrides the AI choice if the chosen direction leads to death and an alternative survives longer. |
| **Smarter** | Bright green | Every-move veto: before each step, simulates `ALWAYS_LOOKAHEAD_STEPS_A` (100) ticks ahead for the AI's chosen direction. If that path leads to death, checks all other valid directions and picks the best survivor. Normal food-seeking is otherwise untouched. |
| **AGI** | Purple | Same as Smarter but with a deeper simulation of `ALWAYS_LOOKAHEAD_STEPS_B` (200) ticks. Slower but catches traps much further into the future.

The veto logic in all lookahead modes is **conservative by design**: if the AI's current choice already survives the full simulation window, no override happens. The lookahead only acts when it detects an incoming death and a better alternative exists.

### Reset button
Immediately starts a new round (current score is still recorded to stats).

---

## Statistics

All values are comparable regardless of grid size; each stat shows the cell count (W×H) of the grid it was achieved on in parentheses.
The local storage key for reset is `opticode-snake-ai-stats`.

| Stat | Meaning |
|------|---------|
| **Max score** | All-time highest score (food eaten in one round) |
| **Max cells filled** | Highest percentage of the grid covered by the snake (length ÷ total cells) |
| **Max round length** | Longest the snake has ever grown (in cells) |
| **Min cells filled per score** | Lowest ratio of snake length to score. Lower is better — it means the snake reached food faster with less wandering |

The **Last round** section shows the same four stats for the most recently completed game.

---

## Can the bot be improved?

Yes — but it is genuinely hard, and the tension is interesting.

The snake problem on grid graphs is **NP-hard**. There is no known algorithm that can guarantee optimal play (filling the grid completely every round) in polynomial time. The best theoretical solution is a **Hamiltonian cycle** — a pre-computed path that visits every cell exactly once — but constructing one that also takes useful shortcuts toward food, for arbitrary grid sizes, without becoming predictable and visually dull, is a real engineering challenge.

The current bot uses weighted Dijkstra pathfinding, a strict flood-fill safety check, tail chasing as a fallback, cycle detection, and optional forward simulation. It performs well and looks good doing it.

**This is the core tension**: a bot that never dies is boring to watch. A bot that plays recklessly dies too often. The sweet spot — long interesting rounds with varied paths and occasional dramatic near-misses — is what this project aims for aesthetically, not just numerically.

---

## Contributing

This is an open source project and PRs are very welcome. If you have an idea for improving the bot's score *without* making it visually less interesting.

---

## Technical breakdown

### Grid and rendering

The grid is computed dynamically from the viewport at startup and on resize. On screens ≥ 1920 px wide, cells are 40×40 px; otherwise 20×20 px.

---

### Pathfinding — primary strategy

The bot's first priority is reaching food. It uses a **weighted Dijkstra search** (`pathToTargetAvoidingEdges`) that assigns a higher cost to cells within one step of a wall. This produces paths that naturally favour the interior of the grid, keeping the snake away from edges where the risk of being cornered is highest.

```
cost(cell) = AI_EDGE_STEP_COST (2)   if edgeDistance(cell) ≤ 1
             1                        otherwise
```

where `edgeDistance` is the minimum distance to any wall. The shortest-cost path through interior cells is taken.

---

### Safety check — the space requirement

Before committing to the Dijkstra path, the bot simulates the move and applies two checks:

1. **Tail reachability** — after the move (including simulated growth if food is eaten), can the head still reach the tail via BFS? If not, the path is rejected.

2. **Flood-fill space** — the number of cells reachable from the new head position must satisfy:

```
flood_fill(new_head) ≥ snake_length × SPACE_MARGIN
```

`SPACE_MARGIN = 1.5` by default. Raising it makes the bot more conservative at the cost of longer routes to food.

---

### Fallback — tail chasing with body spread

When no food path passes the safety check, the bot switches to a survival strategy:

**Tail chasing** — the bot computes a BFS path toward its *own tail*. Because the tail moves away one cell per tick, the snake perpetually approaches but never quite catches it, effectively tracing its own body in a loop. This guarantees that open space is always being created ahead of the head.

**Body spread scoring** — each candidate move is scored on how far it is from the body:

```
spread(pt) = Σ min(manhattan(pt, body[i]), 5)   for i = 1..min(N, 20)
```

The final fallback score per candidate move is:

```
score = flood_fill
      + spread × AI_SPREAD_MULT (2)
      + (is_tail_path   ? grid_size × AI_TAIL_BONUS_FACTOR (0.2) : 0)
      + (food_reachable ? grid_size × 0.2                        : 0)
      - (near_edge      ? grid_size × AI_EDGE_PEN_FACTOR (0.25)  : 0)
```

Moves where the tail is reachable compete in a separate bracket (`bestSafePt`) and are always preferred over moves where it is not (`bestAnyPt`).

---

### Cycle detection — loop escape

The bot tracks a rolling buffer of recent head positions (`headHistory`). Each tick it checks whether the last `snake_length` positions exactly repeat the `snake_length` positions before them. A perfect repeat is a mathematical proof that the snake is locked in a closed tail-chase loop.

When a loop is detected:
- The history is cleared
- The bot bypasses the flood-fill safety requirement and forces a direct BFS path toward food

This handles the edge case where the snake is densely packed (flood fill always fails the `SPACE_MARGIN` check) and has settled into a stable cycle that would otherwise run forever. The history is also cleared on every food eat, since eating food means genuine progress was made.

---

### Lookahead simulation

All lookahead modes share the same **veto logic**: the AI's normally-chosen direction is simulated first. If it survives the full lookahead window, nothing changes. Only when the simulation detects death does the bot simulate alternatives and switch to the longest-surviving one.

The simulation (`simulateLookahead`) mirrors the exact two-step AI — Dijkstra-to-food then survival scoring — tick by tick from a given state, returning the number of steps survived.

**Targeted mode (`LOOKAHEAD_STEPS`)** — fires only in survival mode (when the Dijkstra food path failed), and only when the snake hits its own body head-on with both perpendicular turns free. Simulates left and right and vetoes the chosen direction if it leads to an earlier death.

**Always-on mode (`ALWAYS_LOOKAHEAD_STEPS`)** — fires every tick regardless of mode. Simulates the AI's chosen direction; if it leads to death, checks all three other valid (non-reverse) directions and picks the best survivor. This preserves the food-seeking behaviour entirely — the lookahead only intervenes for dangerous moves.

---

### Tunable parameters

All parameters live at the top of the `<script>` block.

| Parameter | Default | Effect |
|-----------|---------|--------|
| `SPACE_MARGIN` | `1.5` | Flood-fill multiplier for the safety check. Higher = more conservative |
| `AI_EDGE_STEP_COST` | `2` | Dijkstra cost for near-wall cells (interior = 1). Higher = stronger wall avoidance |
| `AI_EDGE_PEN_FACTOR` | `0.25` | Survival-mode edge penalty = `grid_size × factor` |
| `AI_TAIL_BONUS_FACTOR` | `0.2` | Survival-mode tail-chase bonus = `grid_size × factor` |
| `AI_SPREAD_MULT` | `2` | Multiplier for the body-spread score in survival mode |
| `LOOKAHEAD_STEPS_A` | `0` | Steps simulated in Normal mode (0 = disabled) |
| `LOOKAHEAD_STEPS_B` | `100` | Steps simulated in Smart mode (targeted perpendicular veto) |
| `ALWAYS_LOOKAHEAD_STEPS_A` | `100` | Steps simulated in Smarter mode (every-move veto) |
| `ALWAYS_LOOKAHEAD_STEPS_B` | `500` | Steps simulated in AGI mode (deeper every-move veto) |

---

### Complexity

The snake problem on grid graphs is **NP-hard** (proven in the literature). No polynomial-time algorithm can guarantee optimal play for all configurations. The strategies above are heuristic — they perform well in practice but cannot guarantee a perfect score.

The per-tick computational cost (Normal mode):
- Dijkstra: O(N log N) where N = grid cells
- BFS tail check / flood fill: O(N) each, run at most 3–4 times per tick
- Body spread: O(20) = constant

In lookahead modes each simulated tick runs the same AI, so cost scales with `lookahead_steps × O(N log N)`. At typical grid sizes (800–2000 cells) Normal mode completes in under 1 ms. Smart mode adds a few ms only at the specific perpendicular decision points. Smarter and AGI modes run a full simulation every tick and are noticeably slower at high speeds — this is expected and intentional.

---

*Built by OptiCode. Single-file, zero dependencies, runs anywhere.*
