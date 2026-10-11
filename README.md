# Chess in ZZ

A complete, playable chess engine written in pure ZZ plus registry
packages (`rand` for variety draws, `ask` for the menu/move prompts,
`tui` + `toast` for the upcoming board frontend) — no plugins, no
hand-written natives. It exists to prove the language can build real
things: a `zz.toml` project, eight modules, structs, recursion,
copy-make search, registry + path dependencies, and an interactive
TUI with graceful non-TTY fallbacks.

```
src/main.zz          CLI app: menu, play/watch, gauntlet, analyze, FEN
src/engine/          reusable core — no UI imports, any frontend works:
  board.zz           position model, setup, display, move names
  moves.zz           legal movegen (castling, ep, promotion), attacks,
                     check detection, make/null-move, pin-aware filter
  fen.zz             FEN import/export (Option, never throws)
  san.zz             SAN display + coordinate/castle parsing
  eval.zz            tapered eval (fast leaf path + full structure:
                     outposts, 7th-rank rooks, king-ring danger)
  ai.zz              negamax: PVS, LMR, null-move, futility, MCP,
                     check + single-reply extensions, deep quiescence,
                     MVV-LVA
  person.zz          rated personalities + ID w/ clock (search_root)
  book.zz            19-line opening book
src/uci/             UCI protocol interface (uses engine only)
  uci.zz             position/go/bestmove, Skill/Elo options
src/tui/             terminal board frontend (uses engine only)
docs/plan-smart-engine.md  strength roadmap (what ZZ value semantics
                           allow, what they rule out)
```

Rule: `engine/` imports std + `rand` only — never `ask`, `tui`,
`toast`, or any interface module. Frontends (`main`, `uci`, `tui`)
depend inward on `chess.engine.*`. Dotted dependency imports
(`import tui.screen`, `import toast.win`) work since the toolchain
fix — see notes.

Run with the dev toolchain (`d_zz`). First fetch the dependencies:

```bash
d_zz install
d_zz run src/main.zz perft     # perft(1..3) vs 20 / 400 / 8902
d_zz run src/main.zz ai        # Club vs Club, 40-ply demo
d_zz run src/main.zz play      # human (white) vs Club, moves like e2e4
d_zz run src/main.zz gauntlet  # Novice vs Club x4, Elo table
d_zz run src/main.zz analyze   # Expert top-3 over startpos (FEN optional)
d_zz run src/main.zz uci       # UCI loop for GUIs / Cutechess
d_zz run src/main.zz tui       # board UI, you (white) vs Club
d_zz run src/main.zz fen "r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1"
d_zz run src/main.zz           # interactive menu
```

The menu is an `ask` select list: pick Play vs engine in the board UI
or plain text (then your color and which personality), Watch engine vs
engine (pick both), or Self-test. Text moves are typed as `e2e4`
(`e7e8q` for promotion, `O-O` / `O-O-O` for castles, `hint` for a
coach suggestion, `resign` to quit, confirmed). Engine replies print
in SAN with `[book]` or think time.
The first moves of every game come from the built-in 19-line book.
On non-TTY stdin the prompts fall back to numbered/line input, so the
whole flow is pipe-testable. (If stdin is *closed*, `ask` retries
forever — upstream issue, Ctrl-C out.)

## UCI

`d_zz run src/main.zz uci` speaks Universal Chess Interface:
`uci` / `isready` / `setoption name Elo|Skill Level` / `ucinewgame` /
`position [startpos|fen ...] [moves ...]` / `go [depth|movetime|
wtime/btime/winc/binc|infinite]` / `d` / `quit`. Replies carry
`info depth/score/time/pv` and `bestmove`. Point Cutechess at the
native binary for rated games:

```bash
d_zz build -o /tmp/chess src/main.zz
printf 'uci\nisready\nposition startpos\ngo depth 6\nquit\n' | /tmp/chess uci
```

## Config: ~/.config/chesz/config.json

First launch creates it with defaults (never overwrites):

```json
{
  "pieces": "letters"
}
```

`pieces` picks the set: `letters` (default — works everywhere),
`ascii` (character art), `glyphs` (Unicode). `CHESS_PIECES` env wins
over the file when both are set.

## Pieces: ASCII art, glyphs, letters

Three tiers, worst first: **ASCII character art** by default (7×4
stroke sets, byte-clean for layout math — works on every terminal
ever made), **glyphs** via `CHESS_PIECES=glyphs`, **letters** via
`CHESS_PIECES=letters` or no color. The kitty-graphics PNG experiment
is fully removed: silent failures (blank boards) are worse than an
honest fallback, and no terminal in reach rendered them. Raw mode is
global, not refcounted: lobby re-arms it every turn because
game/watch restore cooked on exit — otherwise arrows echo as garbage
mid-menu.

## Board UI

`src/tui/` is a retained frontend on the same engine. Launching with
no args opens floating palettes (play / hotseat / watch / quit →
clock → color → engine),
then the big board: centered 7×2 lichess-brown squares with a closed
border frame (truecolor, 256-color, and basic16 tiers with a plain-ASCII
fallback), hand-designed pixel pieces (7×4 art rasterized to
half-block pairs — same silhouette both sides, ink color reads the
side), cursor
(arrows/hjkl), select with Enter (targets light up), promotion
picker, coach hint (`?`), undo (`u`, one ply in hotseat, two vs the
engine), resign (`q`), eval bar (`e`,
hidden by default). Checks pop
top-right via `toast.top`; game-over and side panels never show stale
to-move lines. Screen ownership via `tui.screen`, key decoding via
`alias tui.keys` (full `ask` entry collides — zaidejjo/zz#380). Watch
mode spectates engine vs engine with a quit poll between moves — pick
a different rating per side (white Master, black Joker, anything);
`e` toggles the same eval bar there.

Hotseat runs two humans on one board (either color to move, takebacks
cost a single ply). The clock palette offers Casual (no clock), Blitz
5+0, or Rapid 15+10: remaining time sits in the side panel, the mover
pays think time plus gains the increment, and expiry ends the game
(`1-0 White wins on time`). No live ticking — the blocking key reader
cannot redraw mid-thought — so the clock settles when each move lands.

Every board-UI game ends with a printed report: result, move count,
per-side accuracy (100 − 3 × win-points-dropped per move, from the
cached eval history), each side's worst move, and the final eval line.
Quit mid-game and the report says `unfinished` instead.

Every played game auto-saves PGN to `~/.config/chesz/games/` (headers
+ movetext + result, millisecond filename, never overwritten).
`d_zz run src/main.zz pgn FILE` imports one back: tags skipped, move
numbers/results/NAGs/comments dropped, each token matched through SAN
— it prints every move, then the final board and result, stopping at
the first token it cannot name.

The lobby's Load PGN entry opens a file browser instead of a path
prompt: directories first, only `.pgn` shows, arrows/hjkl move, Enter
opens, Backspace climbs, Esc cancels. It reopens at the remembered
directory — every visit persists `last_dir` in
`~/.config/chesz/config.json` beside the pieces setting (never
clobbered on write). After a good parse the game replays onto a live
board and continues from the final position: pick a side, engine and
clock as usual, with full eval history, report and re-save.

All option picking (lobby, promotion, resign, game-over) goes through
`src/tui/pick.zz`: centered `win.box_at` drawing plus a decoded loop
(arrows/hjkl/home/end/enter/esc). The toast `palette`/`show` widgets
read raw bytes but compare decoded codes, so arrows arrive as ESC and
cancel — unusable here; `toast.top`/`flash` (timed, keyless) and pure
`box_at` stay.

The eval bar is hidden until `e`: a 16-tall strip beside the board
plus an `Eval: +0.65 · White 64%` panel line (mate scores show as
mate/mated). Iterative-deepening 1→2 under a 200ms clock re-runs on
every *position* — depth 1 always lands instantly, depth 2 refines
when time allows — while cursor movement never searches. The strip
fills in tenths of a row (░▒▓ waterline) so it slides instead of
jumping ranks. It runs
synchronously on purpose: ZZ green tasks exist (`task.spawn`) but
`try_join` never observes completion and two in-flight spawns hang
(zaidejjo/zz#398), so no background worker can drive the UI until
upstream fixes polling.

## Personalities

One search, many strengths: depth + clock set the ceiling, `variety`
(a centipawn window around best) and `blunder` (random-move rate)
bring it down to human. Forced mates are never "varietied" away.

| Engine | Elo | Depth | Move budget | Variety | Blunder |
|--------|-----|-------|-------------|---------|---------|
| Joker  | ~400 | — | — | — | pure chaos (uniform legal) |
| Novice | 800 | 1 | 200 ms | 150 cp | 8% |
| Club   | 1200 | 2 | 500 ms | 60 cp | 2.5% |
| Expert | 1600 | 3 | 1.5 s | 25 cp | 0.8% |
| Master | 2000 | 4 | 3 s | 0 | 0 |

Measured reality (VM, dev toolchain): depth-1 ≈ 0.3 s/move, depth-2
≈ 20 s/move, depth-4 ≈ 30+ min. **Expert/Master are native-only**
(the AOT binary runs ~20x: depth-2 ≈ 1 s, depth-4 playable).
Novice/Club are comfortable on the VM. Budgets above are honored by
iterative deepening — worst case the engine falls back to depth 1.

Iterative deepening guarantees a move even when the clock fires mid
thought (worst case: the depth-1 reply). Root scoring abandons its
tail at the deadline, so overshoot is one move, not one depth. Joker is the "random rating":
every legal move equally likely, every game a new disaster.

## Draws

Fifty-move rule, threefold repetition (history keys carry side +
board, 12 deep), and dead-material detection (bare kings, single
minor) all end games with explicit score lines. En-passant, castling
rights, and both clocks ride through FEN and `make`/`null_move`.

Native build (same sources, AOT through clang):

```bash
d_zz build -o /tmp/chess src/main.zz
/tmp/chess perft
```

## VM vs native (perft 1-3, startpos)

| engine | time |
|--------|------|
| VM (`d_zz run`) | ~10 s |
| native AOT | ~0.5 s |

The gap is the tree-walking interpreter plus full copy-make movegen per
node — expected at this stage, and exactly what the two-engine design
is for: iterate on the VM, ship the native binary. See the tracking
issue for the VM/AOT performance roadmap.

## Notes for ZZ authors

- Locals/params must not reuse their own module's name (e.g. no local
  called `moves` inside `moves.zz`, no param called `fen` in `fen.zz`):
  it shadows the module namespace at runtime and helper resolution
  dies with `undefined method` while `zz check` stays green. See the
  comment atop `src/moves.zz`. Filed as zaidejjo/zz#376.
- `tests/` files must import siblings as `chess.<module>` (bare
  `import board` resolves relative to `tests/` and fails); `src/`
  files use bare imports. The `ask` package instead nests a test
  sub-package with per-module path deps — same effect, more files.
- Dotted imports into *dependencies* (`import tui.screen`) now resolve
  (toolchain fix for zaidejjo/zz#375 — verified). Only entry files
  were reachable before. `tests/` files import engine siblings as
  `chess.engine.<module>`; `src/engine/` files use bare relative
  imports; interface modules (`main`, `uci`, `tui`) use the
  `chess.engine.*` / `chess.uci.*` form.
- Top-level statements cannot call functions defined later in the file;
  define helpers first (all files here do).
- Match guards take pattern bindings, literals, and operators only;
  complex tests belong in the arm body (see `docs/syntax.md`).
- ZZ arrays/maps/structs are value types: threading mutable search
  state (transposition table, killers) through the recursion either
  loses updates or copies per node. The engine uses stateless-only
  selectivity (PVS, LMR, null-move, futility); see
  `docs/plan-smart-engine.md`.
