# ♞ Chesz

Chess in the terminal — a real engine behind a beautiful board UI, written in ZZ.

## Install

```bash
zz install --path chesz
chesz
```

That's it — no cloning. Hacking on the source instead?

```bash
git clone <repo> && cd chesz
zz run src/main.zz
```

## Play

Launch `chesz` and the lobby offers everything:

| Entry          | What                                    |
|----------------|-----------------------------------------|
| Play vs engine | You vs one of five personalities        |
| Two humans     | Hotseat on one board                    |
| Load PGN game  | Browse `.pgn` files, continue playing   |
| Puzzles        | Mate-in-1 pack, three tries each        |
| Board theme    | 15 built-ins + your own files            |
| Watch          | Engine vs engine, any pairing           |

Before a game you pick a clock — Casual, Blitz 5+0, or Rapid 15+10.
After it you get a report: result, accuracy per side, worst moves,
final eval, your record, and your ladder rating. Step through the
finished game first (Review), or go again instantly with flipped
colors (Rematch).

## Board keys

| Key          | Does                              |
|--------------|-----------------------------------|
| arrows/hjkl  | Move the cursor                   |
| Enter/Space  | Select a piece · play the move    |
| Click        | Select a piece · play the move    |
| Type         | Fuzzy-filter any option list        |
| ?            | Coach hint                        |
| u            | Take back (1 ply hotseat, 2 vs AI)|
| e            | Eval bar on/off                   |
| q            | Resign · quit                     |

Everything else is in `chesz --help`.

## Engines

One search, five strengths. Weaker ones pick inside a score window
and occasionally blunder; mates are always played. Adaptive reads your
ladder and plays at your level.

| Engine | Elo  | Depth | Move budget |
|--------|------|-------|-------------|
| Joker  | ~400 | —     | chaos       |
| Novice | 800  | 1     | 200 ms      |
| Club   | 1200 | 2     | 500 ms      |
| Expert | 1600 | 3     | 1.5 s       |
| Master | 2000 | 4     | 3 s         |
| Adaptive | you | your band | 800 ms |

Expert and Master want the native binary (`zz build -o chesz src/main.zz`
runs ~20x the VM). Your lobby ladder starts at 800 and moves K=32 per
rated game.

## Commands

```
chesz                  lobby (the good stuff)
chesz play             text game vs Club
chesz tui [white|black] board game vs Club
chesz ai               Club vs Club demo
chesz gauntlet         Novice vs Club x4, Elo table
chesz analyze [FEN]    Expert top-3 over a position
chesz fen FEN          inspect moves, checks, draws
chesz pgn FILE         replay a PGN file
chesz perft            movegen self-check
chesz uci              UCI loop for GUIs and bots
chesz --help           this list in the terminal
```

Text moves type like `e2e4` (`e7e8q` promotes, `O-O` castles,
`hint` coaches, `resign` quits).

## Files

All under `~/.config/chesz/`:

| File          | What                                              |
|---------------|---------------------------------------------------|
| `config.json` | Pieces (`letters`, `glyphs`, `ascii`), theme (15 built-ins below), last folder |
| `stats.json`  | Your ladder + W-L-D vs every engine               |
| `games/`      | Every played game auto-saved as PGN               |

Boards: lichess, green, slate, blue, catppuccin, latte, tokyo-night,
dracula, nord, gruvbox, solarized-dark, solarized-light, monokai,
everforest, rose-pine — lobby Board theme entry, config, or CHESS_THEME. Your own live in
~/.config/chesz/themes/<name>.json:

```json
{ "dark": "#181825", "light": "#f5e0dc" }
```

Set pieces once and forget it:

```json
{ "pieces": "letters", "theme": "green" }
```

## UCI

`chesz uci` speaks full UCI — `position`, `go`, clocks, `Skill Level`
/ `Elo` options, `info` + `bestmove`. Point Cutechess at it.

## Develop

```bash
zz check src/main.zz   # type-check
zz test                # suite (VM; --engine=native also green)
zz build -o chesz src/main.zz
```

Legal chess throughout: castling, en passant, promotion, draws by
fifty-move rule, threefold repetition, and dead material.
