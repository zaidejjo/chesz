# Smart Engine Plan — strength, speed, personalities

Goal: much smarter + much faster, with rated personalities (800 → 2000)
plus a chaos Joker. UI stays parked until upstream fixes zaidejjo/zz#375.

## 1. Where strength comes from (in priority order)

1. **Eval quality** — tapered piece-square tables + pawn structure +
   bishop pair + rook files + king shield. Biggest Elo lever at fixed depth.
2. **Quiescence** — done. Stops blundering hanging pieces at the horizon.
3. **Selectivity** — PVS + LMR + null-move + futility. ~2-3x effective
   speed: same time, deeper.
4. **Time control** — iterative deepening with a move budget. Stronger
   than any fixed depth: easy moves instant, hard moves use the clock.
5. **Variety model** — personalities pick inside a score window +
   blunder rate. One search, many ratings.

## 2. What ZZ value semantics rule out (documented, not attempted)

Arrays/maps/structs copy on write. Threading a transposition table or
killer/history tables through the recursion either loses updates
(children copies die) or copies kilobytes per node. So: **no in-search
TT, no killers, no history**. What survives: everything stateless —
PVS, LMR, null-move, futility, hash-move-from-last-ID-iteration
(root only, one int). The native binary is ~20x the VM anyway
(perft 10s → 0.5s): ship native, iterate on VM.

## 3. Modules

- `src/eval.zz` (new) — tapered eval. Formula piece-square values
  (== Michniewski tables within a few cp, zero table allocs — the VM
  can't afford 768-int literals per leaf), phase by material
  (N=B=1 R=2 Q=4, /24) driving the king corner-to-center swing,
  bishop pair +30, rook open +12 / semi +6, passed +(15+8/advance),
  isolated −12, doubled −10, king shield +6/pawn (MG-weighted),
  tempo +8. `evaluate(p)` keeps the side-to-move sign contract.
- `src/person.zz` (new) — `Engine{name, elo, depth, movems, variety,
  blunder, joker}` + five presets (Joker ~400, Novice 800, Club 1200,
  Expert 1600, Master 2000) + `best_move_for(eng, p, hist)`.
- `src/ai.zz` — ID loop with `time.now_ms()` budget between depths,
  PVS (null window + re-search), LMR (quiet, idx≥4, depth≥3 → −1, more
  at depth≥5), null-move (R=2, depth≥3, not in check, eval≥beta),
  futility (depth==1, stand+120≤alpha → loud only), move-count pruning
  (depth≤3, late quiets far below alpha), single-reply extension
  (capped), hash move first from previous iteration. `best_move` kept
  as compat wrapper; `search`/`order_moves`/`seen_before` are pub.
  (Aspiration deferred: needs exact-score re-search plumbing; MCP
  covers most of it.)
- `src/person.zz` — `Engine{name, elo, depth, movems, variety,
  blunder, joker}` + five presets (Joker ~400, Novice 800, Club 1200,
  Expert 1600, Master 2000). `search_root` runs one ID climb and
  returns moves+scores+depth (shared by play, `analyze`, UCI);
  `pick_in_window` draws uniformly inside the variety window.
- `src/book.zz` — 14-line opening book keyed by placement+side,
  consulted by all play paths (not UCI).
- `src/uci.zz` — `position`/`go`/`bestmove`, Skill/Elo options,
  `info` lines, closed-stdin guard.
- `src/san.zz` — plus `parse_coord` (shared coordinate/castle
  parser; `main`'s dup deleted).
- `src/moves.zz` — plus `null_move`.
- `src/board.zz`, `src/fen.zz` — untouched.
- `src/main.zz` — personality picker, `play`/`ai`/`gauntlet` (Elo
  table via piecewise-logistic), `analyze` top-3, `hint` coaching,
  `fen` inspector, `uci` loop.

## 4. Personalities

| Engine | Elo | Depth | Move ms | Variety | Blunder |
|--------|-----|-------|---------|---------|---------|
| Joker  | ~400 | —   | —       | —       | pure random legal |
| Novice | 800 | 1     | 200     | 150cp   | 8%      |
| Club   | 1200 | 2    | 500     | 60cp    | 2.5%    |
| Expert | 1600 | 3    | 1500    | 25cp    | 0.8%    |
| Master | 2000 | 4    | 3000    | 0       | 0       |

`variety`: uniform draw via `rand` among root moves within the window
of best (mate scores excluded from the window — never "variety" away
a forced mate). `blunder`: per-mille roll for a uniform-legal pick.
Seeds from `rand.auto_seed()` in play; tests assert properties
(legality, mate-found), never exact moves.

## 5. Verification

- Mate suite: mate-in-1 (3 positions incl. back-rank + smothered
  shape), mate-in-2 (1), all Masters must find the key move.
- TT-free parity: PVS/LMR/null-move must return the SAME best move as
  plain alpha-beta on a fixed suite (correctness of reductions).
- Personality properties: Joker 20/20 legal; Master finds mate-in-1.
- Speed: perft + `ai` demo time before/after in README table.
- Gauntlet: Novice vs Club (2 games, 60-ply cap) finishes, table prints.
- Full `d_zz test` green, perft 20/400/8902 intact, zero warnings.

## 6. Order of work

1. eval.zz + rewire ai (tests: symmetry, known values, mate-term eval)
2. PVS + LMR + null-move + futility + parity tests
3. ID + time control + person.zz + menu/gauntlet
4. README numbers, commit

## 7. Build log (0.2.0)

- Tapered formula eval landed; startpos scores exactly +8 (tempo).
- `fast()` leaf eval (no census/structure): depth-2 search 95 s → 48 s.
- Pin-aware `gen_legal` (unpinned non-king moves skip copy-make;
  ep/castles always tested): depth-2 → 21 s, perft(3) 10 s → 8 s.
- Single-reply extension (capped), move-count pruning, hoisted
  static eval. Aspiration deferred (MCP covers most of it).
- `search_root` refactor (moves+scores+depth) shared by play,
  `analyze`, UCI. `parse_coord` unified in san.zz.
- 14-line book, `hint` coaching, `analyze` top-3, Elo gauntlet
  (piecewise-logistic, Novice vs Club ×4).
- UCI: handshake, Skill/Elo options, position/go/clocks,
  `info`+`bestmove`, closed-stdin guard. Verified against piped
  commands end to end.
- Full suite pending on VM (depth-4 Master takes 32 min — native-only
  by design; mate test uses a depth-2 solver).
