# ThinkLang — a compact symbolic reasoning format for LLMs.
_Proudly built at [Alias Robotics](https://aliasrobotics.com/) ❤️_


ThinkLang replaces verbose natural-language with structured symbolic blocks, cutting token usage (especially when reasoning phases and rules are stripped out).
Preserves reasoning fidelity, and _possibly_ improves downstream tasks.

*Not benchmarked, only minimally tested with Opus 4.6. Use at your own risk, and feel free to contribute!*


## Quick start

Copy `thinklang.md` into your LLM system prompt, user message, or Claude Code skill directory.
Prompt with pseudocode and math formalisms to save further tokens.

## Format

Responses use ordered blocks:

| Sym | Purpose              |
|-----|----------------------|
| `?` | task                 |
| `#` | context              |
| `$` | premises (stated only)|
| `>` | reasoning            |
| `>!`| challenge (before =) |
| `=` | conclusion           |
| `@` | action               |
| `~` | gaps / missing info  |

### Operators

`→` then · `|` or · `&` and · `¬` not · `=>` if-then · `>>` therefore · `<<` because · `|>` alt-branch

### Confidence

Every claim ends with `~N` (confidence — low = guess, high = certain, any consistent scale).

## Example

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

### Opus 4.6, high effort, thinking, TL

```
? third sister Amanda's activity

$ room, 3 sisters
$ Anna → reading book
$ Alice → playing chess
$ chess requires 2 players

Alice playing chess → needs opponent
candidates in room: Anna, Amanda
Anna → reading << stated
=> Amanda = chess opponent ~9

! attack candidate
%A: chess opponent ∈ room occupants
|> Alice playing chess online/against computer → Amanda unconstrained
|> "playing a game of chess" could = solo study (puzzles, book)
but: standard reading of "playing a game of chess" → active game w/ opponent ~7
%A count: |%A| = 1 (opponent must be co-located human)

! epistemic gate
|$| = 4, |%A| = 1 → proceed (|%A| < |$|)
but alternative (online/solo chess) is plausible ~4

= Amanda playing chess (w/ Alice) ~7 | Amanda's activity underdetermined ~4

~ "playing chess" ambiguous re: opponent identity; no premise explicitly states Amanda is the opponent
```

## Claude Code integration

```bash
mkdir -p ~/.claude/skills/tl
cp thinklang.md ~/.claude/skills/tl/SKILL.md
```

Then invoke with `/tl` in any Claude Code session.

## License

MIT
