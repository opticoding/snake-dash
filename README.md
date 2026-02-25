# SNAKE DASH

**An ambient snake bot that plays itself — forever.**

---

## What is this?

Snake Dash is a single HTML file you can open in any browser and leave running. There is no player, no goal to complete. Algorithms pilots the snake across the grid continuously, eating, growing, and starting fresh when a round ends — all on its own.

It was built as a **living screensaver**: something beautiful to leave on a second monitor, a digital photo frame, a lobby display, or a TV. Designed to look good in the dark, for any length of time.

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

## Can the bot be improved?

Yes — but it is genuinely hard, and the tension is interesting.

The snake problem on grid graphs is **NP-hard**. There is no known algorithm that can guarantee optimal play (filling the grid completely every round) in polynomial time. The best theoretical solution is a **Hamiltonian cycle** — a pre-computed path that visits every cell exactly once — but constructing one that also takes useful shortcuts toward food, for arbitrary grid sizes, without becoming predictable and visually dull, is a real engineering challenge.

The current bot uses weighted Dijkstra pathfinding, a strict flood-fill safety check, explicit tail chasing as a fallback, and a body-spread heuristic to avoid coiling. It performs well and looks good doing it. Raising the score further usually means being more aggressive about shortcuts — which increases the risk of self-trapping — or implementing a Hamiltonian cycle as a safety net, which tends to make the movement mechanical and repetitive.

**This is the core tension**: a bot that never dies is boring to watch. A bot that plays recklessly dies too often. The sweet spot — long interesting rounds with varied paths and occasional dramatic near-misses — is what this project aims for aesthetically, not just numerically.

---

## Contributing

This is an open source project and PRs are very welcome.

Some directions worth exploring:

- **Hamiltonian cycle with smart shortcuts** — generate a base cycles and allow the bot to skip ahead when it is provably safe to do so, without making movement look robotic
- **Better trap detection** — identify space partitions before entering them, not just after
- **Adaptive `SPACE_MARGIN`** — tighten or relax the safety threshold dynamically based on how full the board is
- **Visual / aesthetic improvements** — trail effects, colour themes, different rendering styles

If you have an idea for improving the bot's score *without* making it visually less interesting, that is exactly the kind of PR worth discussing.

→ [Open an issue or PR on GitHub](https://github.com/OptiCode/dash-snake)

---

## Tuning it

All parameters live at the top of the `<script>` tag inside `index.html`:

| Variable | Default | Effect |
|---|---|---|
| `SPEED_DEFAULT` | `200` | Starting tick interval in ms (lower = faster) |
| `SPACE_MARGIN` | `1.0` | Safety multiplier — raise for more cautious play |
| `EAT_FLASH_ENABLED` | `false` | Screen flash on food intake |
| `SMOOTH_THRESHOLD` | `200` | Below this speed (ms), movement interpolation is disabled |

The speed slider in the UI overrides `SPEED_DEFAULT` at runtime.

---

## Statistics

All values are comparable regardless of grid size; each stat shows the cell count (X×Y) of the grid it was achieved on in parentheses.
The local storage key for reset is `opticode-snake-ai-stats`

| Stat | Meaning |
|------|---------|
| **Max score** | All-time highest score (food eaten in one round) |
| **Max cells filled** | Highest percentage of the grid covered by the snake (length ÷ total cells) |
| **Max round length** | Longest the snake has ever grown (in cells) |
| **Min cells filled per score** | Lowest ratio of snake length to score in any round. Lower is better — it means the snake reached food faster with less wandering |

The **Last round** section shows the same four stats for the most recently completed game.

---

## Technical breakdown

### Grid and rendering

The grid is computed dynamically from the viewport at startup and on resize. On screens ≥ 1920 px wide, cells are 40×40 px; otherwise 20×20 px. 

---

### Pathfinding — primary strategy

The bot's first priority is reaching food. It uses a **weighted Dijkstra search** (`pathToTargetAvoidingEdges`) that assigns a cost of 5 to any cell within one step of a wall, and 1 to interior cells. This produces paths that naturally favour the interior of the grid, keeping the snake away from edges where the risk of being cornered is highest.

The heuristic cost function is:

```
cost(cell) = 1  if edgeDistance(cell) > 1
             5  if edgeDistance(cell) ≤ 1
```

where `edgeDistance` is the Chebyshev distance to the nearest wall. The shortest-cost path through interior cells is taken.

---

### Safety check — the space requirement

Before committing to the Dijkstra path, the bot simulates the move and applies two checks:

1. **Tail reachability** — after the move (including simulated growth if food is eaten), can the head still reach the tail via BFS? If not, the path is rejected. A disconnected head–tail means the snake has partitioned itself into a closed region it cannot escape.

2. **Flood-fill space** — the number of cells reachable from the new head position must satisfy:

```
flood_fill(new_head) ≥ snake_length × SPACE_MARGIN
```

Checking only tail reachability is not enough: a narrow corridor can pass a reachability check because technically you can thread through it — but once the body fills in behind you, there is no room left. The space requirement catches these cases. `SPACE_MARGIN = 1.0` is the minimum; raising it makes the bot more conservative at the cost of longer routes to food.

---

### Fallback — tail chasing with body spread

When no food path passes the safety check, the bot switches to a survival strategy:

**Tail chasing** — the bot computes a BFS path toward its *own tail*. Because the tail moves away one cell per tick, the snake perpetually approaches but never quite catches it, effectively tracing its own body in a loop. This guarantees that open space is always being created ahead of the head. It is the single most effective single-move survival heuristic known for this class of problem.

**Body spread scoring** — each candidate move is scored on how far it is from the body:

```
spread(pt) = Σ min(manhattan(pt, body[i]), 5)   for i = 1..min(N, 20)
```

Higher spread = the proposed cell is physically further from recent body segments. Combined with flood fill and edge penalties, the snake naturally distributes itself across the grid rather than coiling tightly, which is the most common cause of self-trapping.

The final fallback score per candidate move is:

```
score = flood_fill
      + spread × 4
      + (is_tail_path ? grid_size × 0.5 : 0)
      + (food_reachable ? grid_size × 0.2 : 0)
      - (near_edge ? grid_size × 0.25 : 0)
```

Moves where the tail is reachable compete in a separate bracket (`bestSafePt`) and are always preferred over moves where it is not (`bestAnyPt`), providing a two-tier fallback.

---

### Complexity

The snake problem on grid graphs is **NP-hard** (proven in the literature). No polynomial-time algorithm can guarantee optimal play for all configurations. The strategies above are heuristic — they perform well in practice but cannot guarantee a perfect score.

The per-tick computational cost is:
- Dijkstra: O(N log N) where N = grid cells
- BFS tail check / flood fill: O(N) each, run at most 3–4 times per tick
- Body spread: O(20) = constant

At typical grid sizes (800–2000 cells) and speeds (10–400 ms/tick), all bot computation completes in under 1 ms.

---

*Built by [OptiCode](https://github.com/OptiCode). Single-file, zero dependencies, runs anywhere.*
