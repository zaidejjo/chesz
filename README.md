# Chess in ZZ

A complete, playable chess engine written in pure ZZ plus two registry
packages (`rand` for tiebreak draws, `ask` for the menu/move prompts) —
no plugins, no hand-written natives. It exists to prove the language
can build real things: a `zz.toml` project, four modules, structs,
recursion, copy-make search, registry dependencies, and an interactive
TUI with graceful non-TTY fallbacks.

```
examples/chess/
  zz.toml        project manifest (depends on rand ^0.4.0)
  src/board.zz   position model, setup, display, move names
  src/moves.zz   legal move generation (castling, en passant, promotion),
                 attack maps, check detection, make-move
  src/ai.zz      material + centralization eval, negamax with alpha-beta,
                 captures-first ordering, mate scores, repetition filter,
                 random tiebreaks via rand
  src/main.zz    perft self-test, AI vs AI demo, human vs AI
```

Run with the dev toolchain (`d_zz`). First fetch the dependencies:

```bash
cd examples/chess
d_zz install
d_zz run                       # interactive menu (play, watch, self-test)
d_zz run src/main.zz perft    # perft(1..3) vs 20 / 400 / 8902
d_zz run src/main.zz ai       # AI vs AI, depth 2, 40-ply demo
d_zz run src/main.zz play     # human (white) vs AI, moves like e2e4
```

The menu is an `ask` select list: pick Play vs engine (then your color
and engine strength), Watch engine vs engine, or Self-test. Moves are
typed as `e2e4` (`e7e8q` for promotion, `resign` to quit, confirmed).
On non-TTY stdin the prompts fall back to numbered/line input, so the
whole flow is pipe-testable. (If stdin is *closed*, `ask` retries
forever — upstream issue, Ctrl-C out.)

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
  called `moves` inside `moves.zz`): it shadows the module namespace at
  runtime and helper resolution dies with `undefined method` while
  `zz check` stays green. See the comment atop `src/moves.zz`.
- Top-level statements cannot call functions defined later in the file;
  define helpers first (all files here do).
- Match guards take pattern bindings, literals, and operators only;
  complex tests belong in the arm body (see `docs/syntax.md`).
