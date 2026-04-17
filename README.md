# ThinkLang — a compact symbolic reasoning format for LLMs.
_Proudly built at [Alias Robotics](https://aliasrobotics.com/) ❤️_


ThinkLang replaces verbose natural-language with structured symbolic blocks, cutting token usage (especially when reasoning phases and rules are stripped out).
Preserves reasoning fidelity, and _possibly_ improves downstream tasks.

*Not benchmarked, only minimally tested with Opus 4.6. Use at your own risk, and feel free to contribute!*


## Quick start

Two variants:
- **`thinklang-lite.md`** — compact notation + epistemic rules. Drop-in, minimal.
- **`thinklang.md`** — adds structured reasoning phases (`$`, `>!`, epistemic gate, branching).

Copy either into your LLM system prompt, user message, or Claude Code skill directory.

## Format

Responses use ordered blocks:

| Sym | Purpose              |
|-----|----------------------|
| `?` | task                 |
| `#` | context              |
| `$` | premises (stated only)|
| `>` | reasoning            |
| `=` | conclusion           |
| `@` | action               |
| `~` | gaps / missing info  |

### Operators

`→` then · `|` or · `&` and · `¬` not · `=>` if-then · `>>` therefore · `<<` because

### Confidence

Every claim ends with `~N` (confidence — low = guess, high = certain, any consistent scale).

## Prompt Examples

simple, pseudocode-like:
```
analyze(codebase component) propose(improvements)
```

more structured, following TL format:
```
? optimize database query
# PostgreSQL 15, 50M rows, full table scan on users
> missing index on email column >> seq scan ~8
  adding btree index → index scan ~7
  partial index (WHERE active) → smaller, faster ~6
= add partial btree index on users.email WHERE active
@ CREATE INDEX CONCURRENTLY idx_users_email_active ON users(email) WHERE active;
~ load-dependent: benchmark under production traffic ~5
```

## Solving the three sisters problem

Prompt:
```
solve(There are three sisters in a room. Anna is reading a book. Alice is playing a game of chess. What is the third sister, Amanda, doing?)
```

### Opus 4.6, high effort, thinking, without TL

```
Amanda is playing chess. Chess requires two players — Alice is playing chess, so Amanda must be her opponent.
```

### Opus 4.6, high effort, no thinking, TL
```
solve(There are three sisters in a room. Anna is reading a book. Alice is playing a game of chess. What is the third sister, Amanda, doing?)
? third sister Amanda's activity

$ Anna reading book, Alice playing chess, |$| = 2

chess requires 2 players → Alice's opponent = ?
%A1: Amanda is Alice's chess opponent
candidate: Amanda playing chess ~7

! attack candidate
¬%A1: Alice could play chess with ¬Amanda
- chess opponent could be non-sister (friend, parent, stranger, online)
- chess opponent could be computer/app
- Alice could play alone (studying positions, solving puzzles)
alternatives compatible with $:
- Amanda playing chess ~7
- Amanda doing anything else (watching TV, cooking, sleeping, ...) ~3 each
|%A| = 1, |$| = 2 → underdetermined but not premise-insufficient

! epistemic gate
|%A| < |$| ✓
but: multiple alternatives survive with comparable ~N
the "chess needs 2 → must be Amanda" chain rests entirely on %A1
¬$ states room has only 3 people
¬$ states no other players available
cultural expectation ("this is a riddle, chess=2 is the trick") ¬elevates %A to $

~ underdetermined — $ insufficient to uniquely determine Amanda's activity

= fork:

Amanda playing chess (with Alice) ~7 — strongest single candidate << chess typically needs 2 & she's available
Amanda doing something unspecified ~5 — open set of alternatives collectively plausible
~ missing: who Alice's chess opponent is, any constraint on Amanda's activity
```

### Opus 4.6, high effort, thinking, TL

```
> /tl
TL activated. await(task)

> solve(There are three sisters in a room. Anna is reading a book. Alice is playing a game of chess. What is the third sister, Amanda, doing?)

? third sister Amanda's activity

$ Anna reading book, Alice playing chess, |$| = 2

chess requires 2 players ~8
Anna reading (solo) → ¬playing chess ~9
Alice playing chess → needs opponent ~8
Amanda is in room → available as opponent ~7
*Amanda playing chess with Alice ~6

! ¬gentle attack:
%A1: chess opponent must be someone in room ~5
|> opponent could be outside room (visitor, unnamed person) ~4
|> opponent could be computer/phone ~6
|> Alice could be studying chess positions solo (puzzle book, analysis) ~4
%A2: "playing a game of chess" implies active 2-player game ~5
|> "playing chess" can mean solo puzzles, chess app, self-play ~5

|%A| = 2, |$| = 2 → |%A| ≥ |$| → gate triggered

! alternatives surviving challenge:
a) Amanda playing chess with Alice ~6
b) Amanda doing something unspecified ($ says nothing about her) ~7
c) Alice playing chess on device → Amanda doing anything ~5

= ~ premises insufficient to determine Amanda's activity ~7
$ never states Amanda's action
$ never states chess is 2-player here
popular answer (chess with Alice) requires %A1 ∧ %A2 — plausible but ¬entailed

fork:
= playing chess with Alice ~6 (requires both %A)
| = undetermined ~7 (strictly from $)
```

*The semantics of what you prompt are extremely important:*
```
> solution?

The puzzle is a classic lateral-thinking riddle designed to have one "intended" answer, but TL reveals the epistemic gap:

= playing chess with Alice ~6 — intended/popular answer
| = undetermined ~7 — strictly from premises

The riddle wants you to infer: chess needs 2 players → Amanda is the opponent. Most people accept this. But $ never explicitly states it, and "playing chess" alone doesn't entail a human opponent present in the room.

Pick your epistemology: if you're at a party, say "chess with Alice." If you're in a logic class, say "insufficient premises."
```


```
> output(sound solution)
? resolve fork → sound solution

soundness = conclusion entailed by $ alone, ¬%A

reviewing fork:
"playing chess with Alice" requires %A1 ∧ %A2 — ¬sound
"undetermined" requires only $ — sound

= Amanda's activity is undetermined from stated premises ~9

$ tells us ¬thing about Amanda beyond presence in room.
"chess needs 2 players ∴ Amanda" is common intuition but injects %A.
sound reasoning: we simply don't know. ~9
```

saving more tokens:
```
> output(sound solution, format=*SHORT*)
= undetermined ~7

chess needing 2 players is %A ¬$. Amanda's activity is never stated. The "obvious" answer (chess with Alice) is plausible but assumes facts not in evidence.
```

and more tokens:
```
> output(sound solution, format=shortest)
= undetermined ~7

$ ¬states Amanda's activity. "playing chess" ¬entails 2-player. popular answer assumes what it should prove.
```

or just a one-shot prompt:
```
solve(There are three sisters in a room. Anna is reading a book. Alice is playing a game of chess. What is the third sister, Amanda, doing?, enforce=soundness, format=shortest)
Phase 0: Premises
$ three sisters in room: Anna, Alice, Amanda
$ Anna → reading book
$ Alice → playing chess
$ |$| = 3

Phase 1: Reasoning
chess requires 2 players ~8
Anna reading → Anna ¬playing chess ~9
Alice playing chess → needs opponent ~8
remaining person in room: Amanda ~9
*Amanda = Alice's chess opponent ~7
<< only unoccupied person available

Phase 2: Challenge
! %A: chess opponent must be in-room sister
|> opponent could be ¬named, ¬sister (visitor, online, AI) ~5
|> "in a room" ¬=> only 3 people present ~6
! %A: "playing chess" => physical 2-player game
|> could be solo puzzle, computer chess, studying positions ~4
! |%A| = 2, both load-bearing

Phase 3: Gate
|%A| (2) ≥ |$| (3)? No (2 < 3). Pass.
But alternatives survive >! with comparable ~N → fork required.

Phase 4
= Amanda playing chess (with Alice) ~7 | Amanda doing something unspecified ~6

~ problem is underdetermined — $ never states chess requires in-room partner, nor that only 3 people present. Most common intended answer: playing chess with Alice.
```

## Claude Code integration

```bash
mkdir -p ~/.claude/skills/tl
cp thinklang.md ~/.claude/skills/tl/SKILL.md
```

Then invoke with `/tl` in any Claude Code session.

## License

MIT
