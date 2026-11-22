# This project is vibe-coded. Don't read this if you think AI is unacceptable.

# Minesweeper engine

```
npm install
npm test            # all packages
npm run typecheck
npm run dev         # play it locally with live reload
npm run build       # one self-contained file: packages/app/dist/index.html
```

| Package | What it is | DOM |
| --- | --- | --- |
| `packages/core` | Topology, rules, generation, engine, saving and sharing | no |
| `packages/solver` | Deduction, hints, board rating, no-guess generation | no |
| `packages/render` | Theme (number palette), layout (geometry), Canvas renderer | renderer only |
| `packages/app` | The playable page: setup, input, timer, share codes, hints, level editor, library | yes |

Each package depends only on the ones above it. The sections below cover `core`; the
renderer and theme are described at the end.

## Game definition (JSON)

```jsonc
{
  "name": "Optional level title",    // at most 80 characters; the engine ignores it
  "topology": { "type": "square", "width": 30, "height": 16 },
  // { "type": "square", "width": 30, "height": 16, "wrap": true }   edges wrap around (min 3 x 3)
  // { "type": "hex", "width": 16, "height": 16 }                    6 neighbors, odd rows shifted right
  // { "type": "triangle", "width": 31, "height": 16 }               12 neighbors, (x + y) even points up
  // { "type": "custom", "adjacency": [[1], [0, 2], [1]] }           any symmetric graph
  // A square board takes "detects": [[1,2],[2,1],[2,-1],[1,-2],[-1,-2],[-2,-1],[-2,1],[-1,2]] to
  // choose which cells a number counts, as [dx, dy] steps (here a knight's move; up to 8 each way).
  // Any of the three grid types takes "holes": [4, 5, 13] to cut positions (y * width + x) out of
  // the board, for any outline. The remaining cells are numbered 0, 1, 2, ... in reading order.

  "mines": { "count": 99 },                       // random placement (needs generator.seed)
  // { "count": 99, "negative": 20, "maxPerCell": 3 }   20 of the 99 are negative; up to 3 share a cell (max 10)
  // { "count": 40, "types": [{ "value": 2 }, { "value": 5, "weight": 3 }, { "value": -10 }] }
  //                                                    40 mined cells, each a 2, a 5 or a -10; 5s three times as likely
  // { "layout": [3, [17, 2], [42, -1]] }               hand-placed: cell, or [cell, value]

  "rules": {
    "chording": true,                // default true
    "firstClick": "safe",            // "none" | "safe" (default) | "safe-zero"
    "lives": 1,                      // mines you may open before losing (1 to 99)
    "questionMarks": false           // adds "?" to the mark cycle
  },
  "generator": { "seed": 12345 }     // integer, 0..4294967295; seedFromString("text") makes one
}
```

`firstClick` applies to random placement only. Mines are placed on the first reveal, so the
layout is a function of (seed, first click). `"safe-zero"` falls back to `"safe"` on boards
too dense to leave room for an opening.

### Rules in detail

- **Numbers** are the sum of the mine values in neighboring cells: a stack of 3 adds 3, a
  negative mine subtracts 1. A cell holds ordinary or negative mines, never both.
- **Mine types.** A mine's value is what it adds to the numbers around it, from 1 to 10, or -1
  to -10 for negative mines. With `mines.types`, `count` is the number of mined cells and each
  gets a type picked by weight. Which types were dealt depends only on the seed, never on the
  first click, so the mine totals are known from the start; the cells mined are the same ones a
  plain game with that seed would mine.
- **Marks.** The `flag` action steps a closed cell through the mine types in the game
  (`game.mineTypes`): ordinary types smallest first, then negative ones, then `?` if enabled, then
  nothing. For a hand-placed layout the types are the values that appear in it. `?` is only a note and never blocks a reveal, a flood or a chord.
- **Chording** opens the closed neighbors of a number when the flag values around it (plus any
  mines already opened) add up to it. A bare 0 can't be chorded: with negative mines it may be
  +1 and -1 cancelling out.
- **Flooding** spreads only from empty cells: those with no mine in any neighbor. A 0 whose
  mines cancel out is a number like any other and does not flood. `game.touchesMines(cell)`
  tells the two apart.
- **Lives.** Opening a mine costs one life and leaves the mine showing. The game is lost at 0.
- **Winning** is opening every mine-free cell.

## Playing

```ts
const game = createGame(definition);
const events = game.dispatch({ type: "reveal", cell: 0 }); // also "flag", "chord"
game.status; game.stats; game.livesLeft;
game.numberAt(cell); game.mineAt(cell); game.flagAt(cell);
```

## Saving and sharing

```ts
const saved = saveGame(game);         // { version, definition, actions }  (plain JSON)
const same  = loadGame(saved);        // validates, then replays the action log

const text = encodeShare(saved);      // URL-safe string: put it after the "#" of a link
const game2 = loadShare(text);        // or decodeShare(text) to just inspect it
```

Share string = one mode character (`0` raw JSON, `1` deflated JSON, whichever is shorter) plus
base64url. A definition-only link is a save with `actions: []`.

Everything read from a link or file is untrusted: it is validated, size-capped
(including a decompression cap), and fails with `SaveFormatError` or `GameDefinitionError`.

## Stability contract

Seeds and saves live in links people share, so the mine-placement algorithm is part of the
format. Pinned tests in `test/generation.test.ts` and `test/serialize.test.ts` fail if it
changes. A deliberate change means bumping `SAVE_VERSION`.

## Rendering (`packages/render`)

```ts
const layout = layoutFor(game.definition.topology)!;          // geometry for the topology
const renderer = new CanvasRenderer(canvas, game, layout, { cellSize: 32 });
renderer.apply(game.dispatch({ type: "reveal", cell }));      // redraws only what changed
renderer.cellAtClient(event.clientX, event.clientY);          // pointer -> cell id (-1 if none)
```

- **Layout** is the geometry half of a topology: a polygon, a center and a hit test per cell.
  The renderer only talks to that interface, so a new board shape needs a layout and nothing else.
  Square, hex and triangle layouts are built in (`layoutFor(topology)`); a test checks that cells
  touching on screen are exactly the cells the engine treats as neighbors.
- **Theme** is plain data and never part of a save. `numberColors(n, theme.numbers)` applies the
  palette rules: one color for the whole number from -27 to 27 (minus sign included), one color
  per digit beyond that, with the minus sign matching the first digit.
- Numbers are drawn as text in `theme.numbers.fontFamily` (Segoe UI first, then the system font).
- Every number in the palette sits on the same light cell, whatever its sign; per-digit numbers
  get a mid grey. An empty cell is blank, and a 0 whose mines cancel out is drawn.
- `theme.numbers.outline` lists the numbers drawn with an outline (0 and -7 by default: both
  are white). `checkThemeContrast(theme, minRatio)` lists every other number color that falls
  below a contrast ratio on its background.

## Input

`GestureRecognizer` (in `packages/render`, no DOM) turns pointer activity into tap, long press,
pan and pinch. `packages/app` binds it to the canvas:

| | Open | Flag | Open around a number | Move the board | Zoom |
| --- | --- | --- | --- | --- | --- |
| Mouse | click | right click | click the number, or middle click | drag | Ctrl + wheel, or the buttons |
| Touch | tap | hold | tap the number | drag | pinch |
| Keyboard | Space / Enter | F | Space / Enter on the number | arrows, Home/End, PgUp/PgDn | + / - / 0 |

- Flag mode swaps tap and hold.
- Zooming re-rasterizes the board at a whole number of device pixels per cell, so it stays sharp.
- A touch drag the board can't absorb scrolls the page, so a full-screen board never traps you.
- Keyboard movement uses `stepCell(layout, cell, dx, dy)`, which works from geometry alone, so
  future board shapes get keyboard play without extra code.
- Every keyboard move and action is announced through an `aria-live` region
  ("Row 3, column 5: 2", "Opened 12 cells.", "Mine hit. Game over.").

## Solver (`packages/solver`)

```ts
const hint = findHint(game);                 // { cell, value, tier, sources, needs } or null
const rating = rate(definition, firstClick); // { solved, hardestTier, rounds }
const found = generateNoGuess(definition, firstClick, { budgetMs: 4000 });
if (found.definition) createGame(found.definition); // a board logic alone can clear
```

- **It can't cheat.** The solver works from a `BoardView` built from the game's public API:
  open numbers, opened mines and the mine totals. Flags and question marks are the player's
  own notes and are not treated as evidence.
- **Three tiers of reasoning**, tried easiest first, so the tier of a result says how hard the
  position is: (1) one number at a time, plus the mine count; (2) two numbers compared, which
  covers the classic 1-2-1 and 1-1 patterns; (3) every arrangement of mines that fits a
  connected group of numbers, keeping what is the same in all of them.
- **Sound, not complete.** It never claims something that could be false; a test checks every
  claim against brute force on all shapes and rules. Tier 3 gives up on groups too large for its
  work budget, so it can miss deductions on very tangled boards.
- **It handles every rule:** stacked mines, negative mines, lives and all board shapes, because
  it reasons about a cell's value as a range instead of mine-or-not. When only some mine types
  exist it rounds that range to the values a cell can really hold: on a 2-5-10 board, a 4 beside
  two closed cells can only be two doubles.
- **Hints** prefer a safe cell, then a wrong flag, then an unflagged mine. `needs` lists
  unflagged mines the proof leans on, so an explanation never skips a step.
- **No-guess boards** are found by trying seeds until the solver clears one from the first
  click. The result is a definition with its mines written out as a `layout`, so a saved or
  shared no-guess game holds the actual board and replays the same even if the solver changes.
  In the page the search runs in a Web Worker.

## Level editor and library (`packages/app`)

- **`LevelEditor`** (`src/editor.ts`, no DOM) holds a board shape and hand-placed mines, keeps
  the numbers up to date, and turns into a `layout` definition. It exposes a `BoardSource`, the
  small read-only interface the renderer draws from, so the editor board is drawn by the same
  renderer as a game. A level is an ordinary definition: it plays, saves and shares like any other.
- **`Library`** (`src/library.ts`) keeps saved levels and games in the browser's `localStorage`
  as share codes. It is handed its storage, so tests use a stand-in. Everything read back is
  validated, and a browser that refuses storage just gives an empty, read-only library.
- **Session memory.** The current game, the setup and any level being edited are also kept in
  `localStorage`, so closing the page loses nothing. A game code in the address wins over it.
- Storage is per browser: it does not follow you to another device. Codes do.

## Board outlines and themes

- **Holes.** `gridPositions(topology)` gives the grid position of each cell, for turning a cell id
  back into a row and column. `layoutFor` returns a layout cropped to the cells that are left, and
  keyboard steps jump over holes. Nothing else in the engine or solver knows holes exist: a board
  with holes is just a graph with fewer cells.
- **The editor** works on the whole grid (so a cut-out cell can be put back) and converts to the
  engine's numbering when it produces a definition. `BoardSource.isVoid` tells the renderer which
  positions to draw as empty outlines.
- **Theme settings.** `themeSettings(theme)` and `applyThemeSettings(settings)` read and write the
  parts of a theme a player may change: the colors listed in `THEME_COLORS`, the outlined numbers
  and the grid line thickness. The number colors are not among them and `applyThemeSettings`
  never takes them from its input, so a pasted theme can't alter the palette.
- A theme belongs to the viewer, not the game: it is kept in the browser and is not part of a
  game code.

## Detection patterns

- With `detects`, a square board's graph is built from the pattern (`buildPattern`) instead of the
  8 cells around. Everything that follows neighbors follows the pattern: numbers, the flood from an
  empty cell, chording, first-click protection, the solver and hints.
- A pattern may be one-way (`[[1, 0]]`: each cell detects only the cell to its right), so the graph
  is not always symmetric. The engine and solver only ever ask "what does this cell detect"; the
  level editor also keeps the reverse ("what detects this cell") to update numbers when a mine moves.
- **Hexagons** take `detects` too, as `[dq, dr]` steps: `dr` rows down and `dq` cells along the
  row, where a row down leans half a cell right. Unlike column offsets, these mean the same on
  every row, staggered or not (`buildHexPattern` does the conversion).
- **Triangles** take `detects` as `[dx, dy]` steps seen from a triangle pointing up. A triangle
  pointing down is its mirror image, so it uses the pattern with `dy` reversed
  (`buildTrianglePattern`). The ordinary board is the 12-step pattern of touching cells.
- In the page the picker's reach is adjustable: up to 8 cells each way for squares, the engine's
  limit, up to 6 rings for hexagons, and up to 4 rows (8 columns) for triangles.
- `renderer.setRange(cells)` marks cells on the board, used to show what the pointed-at cell detects.

## Weights and per-type patterns

- A step may carry a third number, a **weight** from 1 to 10: `[0, -1, 2]` counts a mine in the
  cell above twice. A number is the sum, over the cells it detects, of weight times mine value.
- **`detectsFor`** gives some mine types a pattern of their own:
  `"detectsFor": [{ "types": [5], "steps": [[1,2],[2,1], ...] }]` detects 5-mines a knight's move
  away while every other type uses `detects` (or the shape's ordinary pattern).
- The graph then carries `edgeKind` and `kinds`; `edgeWeight(graph, k, value)` reads how many
  times a mine of that value counts at an entry of `neighbors` (0 = that kind isn't detected there).
  Ordinary boards have neither field and behave exactly as before.
- **An empty cell proves less on these boards.** It detects no mine, which clears a neighboring
  cell only if every mine type in the game would have been detected there. So the flood, and a
  chord, open only such cells. To keep the start playable, `firstClick: "safe-zero"` opens the
  whole clearing on these boards (it is guaranteed mine-free).
- **The solver** keeps, for every closed cell, the exact set of values it could hold, and treats
  each number as a sum of what each cell would add if it held each value. That covers weights
  and per-type patterns exactly; the brute-force test includes such boards.

## CMY mode

`mode: "cmy"` in a `GameDefinition` swaps mine values for three charges:
1 cyan, 2 magenta, 3 yellow. Charges are unit vectors 120° apart, so one of
each cancels to zero. A cell shows the length and direction of the sum of the
charges it detects:

| Around the cell | Shown |
| --- | --- |
| 1 magenta | magenta 1 |
| 2 cyan | cyan 2 |
| cyan + magenta | blue 1 |
| cyan + yellow | green 1 |
| magenta + yellow | red 1 |
| 2 yellow + 2 cyan | green 2 |
| 2 magenta + 1 yellow | rose √3 |
| 2 yellow + 1 magenta | orange √3 |
| one of each | 0 (white, outlined) |

- `packages/core/src/charge.ts` holds the arithmetic: a charge is stored as
  `[cyan − yellow, magenta − yellow]`, its squared length is `a² − ab + b²`,
  and the label is a whole number or `√n`.
- `game.chargeAt(cell)` returns the charge of an opened cell; `numberAt`
  returns the squared length. `stats.chargeCounts` gives mines per color.
- Every mine counts as one mine on the counter, whatever its color. Flags
  cycle through the kinds of mine in the game. Chording needs flags of the right colors.
- The solver reasons with both components of each charge and the per-color
  totals, so hints and no-guess boards work.
- Weighted pattern cells and per-color detection patterns apply as in
  classic mode.
- CMY colors are fixed hues. They are not part of a theme, and the classic
  number palette is untouched.

### Anti-mines, stacks and color parts

- **Anti-mines** carry the opposite charges: red cancels cyan, green cancels
  magenta, blue cancels yellow. A red mine alone shows a red 1, the same as
  a magenta and a yellow together.
- **Stacks** are two or three mines of one color in one cell. A stack counts
  as that many on the numbers, on the mine counter and in `chargeCounts`.
- A mine's value says both in one number from -9 to 9:
  `1 2 3` one cyan, magenta, yellow; `4 5 6` two; `7 8 9` three; negative
  values are the same for red, green, blue. `chargeColor`, `chargeStack` and
  `chargeValue` convert. Old CMY games (values 1 to 3) load unchanged.
- `stats.chargeCounts` now has six entries: cyan, magenta, yellow, red,
  green, blue.
- **Color parts**: `chargeParts(charge)` splits a charge into amounts of
  cyan, magenta and yellow (at most two are above zero). The renderer's
  `setChargeStyle("parts")` draws those side by side instead of one mixed
  number. It is a display setting only; the game is the same.
- In the page, colors and stack sizes each have their own chance, and the two
  multiply.

## Square root mines (classic mode)

A square root mine (value `ROOT_MINE`, which is 11) adds 1 to every number
that detects it, and those numbers show the square root of their total:

| Around the cell | Total | Shown |
| --- | --- | --- |
| a mine and a root mine | 2 | √2 |
| a mine and two root mines | 3 | √3 |
| one root mine | 1 | 1 |
| three mines and a root mine | 4 | 2 |
| a −4 mine and a root mine | −3 | i√3 |
| a −5 mine and a root mine | −4 | 2i |

- A number is rooted once, however many root mines it detects.
- A root that comes out whole is shown whole, so a 2 can be two mines or
  √4, and a 1 can be √1. `numberAt` gives the shown whole number, or the
  number under the root sign when `rootAt` is true. `shownNumber(sum, roots)`
  in `root.ts` is the rule.
- A total below 0 has an imaginary root, shown as one: i√3 for √−3, 2i for
  √−4, i for √−1. (`numberAt` still gives −3 with `rootAt` true: imaginary
  roots are a matter of display.)
- A root mine counts as one mine on the counter. `stats.rootMineCount` says
  how many there are, in games that can hold them.
- Root mines can be dealt by chance (`mines.types` with `{ value: 11 }`),
  placed by hand, and given a detection pattern of their own
  (`detectsFor: [{ types: [11], ... }]`). They are not allowed in CMY mode.
- Chording works when the flags explain what the cell shows, as the player
  sees it.
- The solver reads a whole number both ways (k with no root mine, or k²
  with one) and settles it when the board does. Hints and no-guess boards
  work.
- A √n number is colored between the palette colors of the whole numbers on
  either side of its root (√2 between 1 and 2). The palette itself does not
  change. The root flag is light blue (`markers.flagRoot`, editable in the
  theme panel) and labelled √.

### Simplified roots

`simplifyRoot(n)` (core) takes the largest square out from under the root
sign: √8 is 2√2, √12 is 2√3, and an imaginary √−8 is shown as 2i√2. The
renderer's
`setRootStyle("simplified")` draws every root that way, in classic and CMY
mode alike; `"plain"` (the default) draws √8. In the page this is the "Show
square roots as" setting, which is remembered between visits. It changes the
display only: the game, the solver and saved games are the same either way.

## Square mines (classic mode)

A square mine (value `SQUARE_MINE`, which is 12) adds 1 to every number that
detects it, and then the number shows its total squared, once for every
square mine it detects, with no limit:

| Around the cell | Shown |
| --- | --- |
| a mine and a square mine | (1 + 1)² = 4 |
| two square mines | 2⁴ = 16 |
| three square mines | 3⁸ = 6561 |
| a −3 mine and a square mine | (−2)² = 4 |
| eight square mines | 8²⁵⁶ |

- With root mines about too, the root is taken first (once) and the first
  square undoes it: √5 squared shows 5, and i√3 squared shows −3.
- Numbers up to `DECIMAL_LIMIT` (99,999) are written out. Larger ones are
  shown as a power c^(2^k), with c not itself a perfect square, so each
  value has one form only and the form gives nothing away. `numberAt` is
  then c and `powerAt` is k. Past 2¹³ the exponent is drawn as 2 to the k.
- `shownNumber(sum, roots, squares)` is the rule; `readingsOf(shown)` lists
  every way a shown number can come about (total, root mines and square
  mines, each as a range), which is what the solver reasons with. A test
  checks that list against the rule exhaustively over a grid of cases.
- A square mine counts as one mine; `stats.squareMineCount` says how many.
  It is drawn as a mine with a square body and no label; its flag is orange
  (`markers.flagSquare`) and labelled x².
- A power's base takes its palette color and background; the exponent's
  digits take the per-digit colors, always outlined.

### Big numbers by place value

The renderer's `setBigStyle("named")` draws numbers too big to write out in
place-value words instead of as powers: 3³² is "1.8530 quadrillion", five
significant digits, rounded, with a mantissa from 1 to 999.99 and the scale
underneath. `placeValue(c, k)` (render) works out c^(2^k) that way without
ever building numbers of more than a couple of thousand digits: past that,
the digits and the scale come from a fixed-point logarithm in BigInt, precise
to well beyond five digits (a test checks it against the exact value).

Scales up to vigintillion use the dictionary names; past that,
`illionName(n)` names the n-th -illion by the Conway–Wechsler system, which
has a name for every one (101 uncentillion, 1000 millinillion). In the page
this is the "Show big numbers as" setting; screen readers hear the same words.
