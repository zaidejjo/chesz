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
| Watch          | Engine vs engine, any pairing           |

Before a game you pick a clock — Casual, Blitz 5+0, or Rapid 15+10.
After it you get a report: result, accuracy per side, worst moves,
final eval, your record, and your ladder rating.

## Board keys

| Key          | Does                              |
|--------------|-----------------------------------|
| arrows/hjkl  | Move the cursor                   |
| Enter/Space  | Select a piece · play the move    |
| ?            | Coach hint                        |
| u            | Take back (1 ply hotseat, 2 vs AI)|
| e            | Eval bar on/off                   |
| q            | Resign · quit                     |

Everything else is in `chesz --help`.

## Engines

One search, five strengths. Weaker ones pick inside a score window
and occasionally blunder; mates are always played.

| Engine | Elo  | Depth | Move budget |
|--------|------|-------|-------------|
| Joker  | ~400 | —     | chaos       |
| Novice | 800  | 1     | 200 ms      |
| Club   | 1200 | 2     | 500 ms      |
| Expert | 1600 | 3     | 1.5 s       |
| Master | 2000 | 4     | 3 s         |

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
| `config.json` | Pieces (`letters`, `glyphs`, `ascii`), last folder |
| `stats.json`  | Your ladder + W-L-D vs every engine               |
| `games/`      | Every played game auto-saved as PGN               |

Set pieces once and forget it:

```json
{ "pieces": "letters" }
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
