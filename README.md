# Pokémon Sims v2

A two-player Pokémon TCG **paper Expanded** simulator for playing complete games between exact decklists and measuring matchup win rates by seat order.

The project is organized around a small rules core, exact deck packages, deck-local strategic brains, reusable policy helpers, reproducible Monte Carlo trials, and strict reporting contracts. Expanded has an enormous combinatorial action space; the architecture exists to keep rules objective, strategy auditable, and expensive reasoning localized.

## Authoritative project contracts

Read these before changing simulator behavior:

1. [`EXPECTED_RESULT_MANIFEST.md`](EXPECTED_RESULT_MANIFEST.md) defines the required end result and output schema. **Do not edit it.**
2. [`official documentation/EN_advanced_manual-2025-transcription-structured.md`](official%20documentation/EN_advanced_manual-2025-transcription-structured.md) is the first rules reference for implemented game mechanics.
3. [`official documentation/compendium-ruling-guidance`](official%20documentation/compendium-ruling-guidance) explains when to use the Pokémon TCG Compendium for interaction-specific rulings.
4. For agents: the local environment should contain `tcg-data-master`, which is the card-data source for exact card text and printing data during development. Card behavior should be tied to exact printings rather than inferred from a card name.

Rules-sensitive code should carry a direct authoritative rules, ruling, or card-data URL beside the relevant implementation when practical. Strategic policy and rules legality must remain separate.

## Required result

Every directory directly representing a deck under `main decks/` is a **main deck package**. Every deck package found recursively under `opposing lists/` is an independent opposing list.

For every:

```text
main deck package × opposing deck package
```

the simulator runs:

- **1,000 complete games with the main deck going first**
- **1,000 complete games with the main deck going second**

The required primary report is one vertically oriented matchup matrix per main deck containing the first-seat win rate, second-seat win rate, combined win rate, and trial count for every exact opposing package. The exact schema and examples live in [`EXPECTED_RESULT_MANIFEST.md`](EXPECTED_RESULT_MANIFEST.md). Where that manifest refers to a deck file, the canonical decklist for a package is its `decklist.txt`.

Every trial is a complete matchup simulation. Item lock, Ability lock, Supporter lock, lead Pokémon, setup races, disruption, recovery, alternate routes, prizing, and matchup-specific interactions arise from the actual simulated board. They are not separate lock-only statistical scenarios.

Detailed diagnostics are encouraged. Useful fields include decisive turn, win condition, lead sequence, lock establishment, failed route, recovery route, reason for loss, major policy decisions, and adjudication reason. These supplement the required win-rate matrix rather than replacing it.

## Repository layout

```text
pokemon-sims-v2/
├── EXPECTED_RESULT_MANIFEST.md
├── README.md
│
├── main decks/
│   └── regidrago-vstar/
│       ├── decklist.txt
│       ├── brain.cpp
│       └── ... optional deck-local .cpp/.hpp helpers
│
├── opposing lists/
│   └── aichi lists/
│       ├── 01-yasunori-kato-shadow-rider-calyrex/
│       │   ├── decklist.txt
│       │   ├── brain.cpp
│       │   └── ... optional deck-local helpers
│       ├── 02-takahiro-ando-vileplume-control/
│       │   ├── decklist.txt
│       │   └── brain.cpp
│       └── ...
│
├── future decks/
│   └── README
│
└── official documentation/
    ├── EN_advanced_manual-2025-transcription-structured.md
    └── compendium-ruling-guidance
```

A **deck package** is a directory containing the canonical pair:

```text
decklist.txt
brain.cpp
```

Additional files may live beside them. A package can have helper `.cpp`/`.hpp` files, local tests, notes, or data if useful. `brain.cpp` is simply the strategic entry point for that exact list.

### `main decks/`

Contains active decks whose matchup matrices are being measured. Each deck package here receives its own independent result matrix.

### `opposing lists/`

Contains independent opponent packages. Discovery is recursive, so organizational directories such as `aichi lists/` do not change semantics.

Do not merge similar lists into one abstract opponent. Small changes in counts, ACE SPECs, Energy, leads, recovery, or lock pieces can change ALS routes and matchup outcomes.

### `future decks/`

Quarantined inputs. Leave this directory alone unless the project owner explicitly promotes a deck into the active simulation set. Its local README is authoritative for that folder.

### `official documentation/`

Local rules references. Use the advanced manual first. Consult the Compendium when the manual does not resolve an interaction cleanly.

## Deck package contract

The package path is the stable identity used by discovery, reporting, replay, and debugging.

For example:

```text
main decks/regidrago-vstar
opposing lists/aichi lists/02-takahiro-ando-vileplume-control
```

The simulator resolves the package's `decklist.txt` for its exact 60 cards and `brain.cpp` for its deck-local policy entry point.

A directory is an active deck package only when it satisfies the expected package contract. Do not recursively treat every `.cpp` or `.txt` file as another deck. Missing `decklist.txt` or `brain.cpp` should be a visible configuration error, not silently ignored during a production run.

Package discovery must exclude `future decks/`.

## `decklist.txt`: PTCGL format plus `Usage:`

Every `decklist.txt` uses the **Pokémon TCG Live deck import/export text format** as its machine-readable card-list portion:

```text
Pokémon: 2
2 Example Pokémon ABC 123

Trainer: 50
4 Example Trainer DEF 45
...

Energy: 8
8 Basic Example Energy ENG 1

Total Cards: 60
```

Card lines preserve quantity, card name, set code, and collector number so the simulator can resolve the exact printing.

The repository extends that format with an optional strategy appendix. Exactly two newline characters after the completed `Total Cards: 60` line may begin:

```text
Usage:
<any amount of text>
```

The raw boundary is:

```text
Total Cards: 60\n\nUsage:\n...
```

Everything from `Usage:` through end-of-file is **commentary, not deck data**.

The deck parser must stop card parsing at that boundary. It must not count, validate as cards, execute, or otherwise interpret anything in the Usage body. Usage may contain arbitrary prose, blank lines, card names, URLs, matchup notes, or lines that resemble PTCGL entries.

A stock PTCGL export ending at `Total Cards: 60` remains valid. Adding or editing Usage never changes the parsed 60-card deck.

### What `Usage:` is for

Usage exists so agents and policy authors understand what the exact 60 is trying to do before implementing its brain.

It can document:

- normal game plan and win condition
- archetype-line-specifics (ALS)
- going-first / going-second opening sequences
- lead Pokémon priorities
- setup races and lock objectives
- resource recursion loops
- attacker roles
- matchup-specific conversion lines
- known traps where a generic card evaluator would choose the wrong objective
- research URLs used to reconstruct the strategy

For example, a Snorlax Stall package should make clear that the primary objective is trapping, Energy denial, recursion, and deck-out/resource exhaustion. A brain that instead optimizes the same 60 cards for damage would produce meaningless matchup data even if every card rule were implemented perfectly.

Usage is guidance, not rules text and not a hardcoded move script. Humans and agents use it to build and audit `brain.cpp`. The production deck parser ignores it.

If tooling exposes Usage metadata separately, it must never override legal-action generation, card text, hidden-information boundaries, or game rules.

### Parser tests

Parser tests should verify that:

- a stock PTCGL export ending at `Total Cards: 60` parses normally;
- the same export plus `\n\nUsage:\n...` produces the same 60 parsed cards;
- arbitrary multiline Usage is ignored by the card parser;
- card-looking lines inside Usage do not add cards;
- malformed deck data before `Total Cards: 60` remains an error;
- exact printing information survives parsing;
- Usage-only edits do not accidentally alter the parsed deck identity.

## `brain.cpp`: deck-local strategy entry point

Every active deck package has a `brain.cpp`.

`brain.cpp` is the **main strategic entry point for that exact decklist**. The simulator uses it to decide what the deck should do next given the legal information and legal actions available in the current game.

Conceptually, a brain behaves like:

```cpp
Action choose_action(
    const PlayerView& self,
    const OpponentPublicView& opponent,
    const LegalActions& actions,
    const MatchContext& context);
```

The concrete interface may evolve, but the ownership boundary should not.

A brain may:

- evaluate the deck's ALS and current route;
- choose opening Active/Bench Pokémon;
- compute dynamic DCI and AMR;
- account for K0/K1 knowledge;
- value lead Pokémon and ordered lead sequences;
- resolve Supporter and connector contention;
- choose search targets, discard costs, gust targets, attacks, retreats, and recovery;
- compare matchup-specific objectives;
- use shared policy helpers;
- use bounded local projection for a genuinely important decision;
- delegate to sibling deck-local `.cpp/.hpp` modules.

A brain may **not**:

- invent legal actions;
- bypass card costs;
- implement generic game rules;
- read unknown opposing hand/deck/Prize information;
- treat engine-omniscient debug information as player knowledge;
- hardcode a synthetic lock that should instead emerge from actual card effects;
- declare a matchup win solely because a strategic position "feels winning."

The rules engine provides legal actions and resolves their consequences. The brain chooses among them.

### Deck-local vs shared strategy

Exact-list strategy belongs in the deck package. Reusable strategic machinery belongs under shared source modules.

For example:

```text
src/policy/common/
  route_analysis.*
  state_evaluation.*
  local_projection.*
  dci.*
  amr.*

src/policy/archetypes/
  regidrago_common.*
  snorlax_common.*
  iron_thorns_common.*
  ...
```

A Regidrago tournament list can call shared Regidrago helpers while its own `brain.cpp` handles its exact Budew count, Goodra/Ranger package, ACE SPEC, lead ordering, or Echoing Horn line.

Do not copy hundreds of lines of identical archetype logic into every package when a shared helper can express the common behavior. Conversely, do not force all lists with the same archetype label through one brain when their exact 60 creates materially different ALS or win conditions.

## Target shared source architecture

Deck brains are colocated with deck packages. The generic simulator grows under `src/`:

```text
src/
├── model/
│   ├── card_id.*
│   ├── card_instance.*
│   ├── zones.*
│   ├── pokemon_in_play.*
│   ├── player_state.*
│   ├── match_state.*
│   └── knowledge_state.*
│
├── rules/
│   ├── turn_engine.*
│   ├── legal_actions.*
│   ├── costs.*
│   ├── continuous_effects.*
│   ├── attack_resolution.*
│   ├── knockout_and_prizes.*
│   └── literal_win_conditions.*
│
├── cards/
│   ├── catalog.*
│   ├── pokemon/
│   ├── trainers/
│   └── energy/
│
├── policy/
│   ├── policy.*
│   ├── common/
│   │   ├── state_evaluation.*
│   │   ├── local_projection.*
│   │   ├── route_analysis.*
│   │   ├── dci.*
│   │   └── amr.*
│   └── archetypes/
│
├── adjudication/
│   ├── adjudicator.*
│   └── matchup/
│
├── simulation/
│   ├── trial_runner.*
│   ├── batch_runner.*
│   ├── rng.*
│   └── pairing_discovery.*
│
├── io/
│   ├── deck_parser.*
│   ├── deck_package.*
│   └── card_data_adapter.*
│
└── reporting/
    ├── game_record.*
    ├── aggregate.*
    ├── matrix_report.*
    └── trace.*

tests/
├── rules/
├── cards/
├── policy/
├── packages/
├── matchups/
└── reporting/
```

Exact filenames may evolve. The ownership boundaries are more important than directory spelling.

## Architectural rules

### The engine owns game mechanics

The rules layer knows how a legal Pokémon TCG game changes state.

It owns turn sequencing, legal-action generation, costs, zones, evolution timing, Energy attachment, Retreat, attacks, damage, Knock Outs, Prize taking, Special Conditions, continuous effects, and objective victory rules.

It should not know that Regidrago wants a Dragon payload in the discard pile or that Evo Vileplume wants Bunnelby + TM Evolution. Those are strategic facts for brains/policy helpers.

Avoid generic-engine branches keyed to archetype names. If an interaction can be expressed through actual cards and current state, model it there.

### Card implementations own printed effects

A card module translates exact printed card text into game effects. It does not decide whether playing that card is strategically desirable.

Shared effects should be reusable. Generic switch, search, gust, draw, discard, damage modification, Ability suppression, attack-cost modification, Item lock, Supporter lock, recovery, and similar primitives should not be reimplemented independently for every deck brain.

### Brains own strategic choices

Brains choose among legal actions.

Policy is where DCI, AMR, ALS, connector domination, Supporter contention, resource preservation, matchup objectives, and bounded projection belong.

A brain may know information legitimately available to its player, including public opponent information and K1 Prize knowledge once earned. It must not inspect hidden opposing information.

### Lead Pokémon are a strategic role

Do not encode `LeadPokemon` as a special rules category.

Goomy, Wobbuffet, Budew, Klefki, Iron Thorns ex, Chimecho, and similar cards are ordinary Pokémon. Their early-game value is recognized by the brain.

Their actual effects operate through the normal rules and board. Ordered plans such as Wobbuffet -> Goomy remain distinct from Goomy -> Wobbuffet because the two orders can change both players' setup graphs differently.

### Locks emerge from the board

Do not represent a real matchup as a synthetic `item_lock = true` scenario.

Vileplume creates Item lock because the correct Vileplume is in play and its Ability is live. Stoutland creates its restriction because it is Active. Wobbuffet suppresses Abilities because it is Active and its effect applies. Goomy taxes attacks because its printed effect is live.

This keeps locks composable with switching, gust, Tool removal, Ability suppression, Stadiums, Knock Outs, and counter-locks.

## Strategic modeling concepts

The simulator should preserve the following concepts when implementing brains and shared policy.

### DCI and UDP

Discard Capable Index (DCI) is dynamic.

`0` means a card should not be discarded under the current state; `1` means it should be discarded immediately if possible. Values in between express relative willingness.

A card can be effectively undiscardable due to play (UDP), singleton value, future resource requirements, matchup-specific needs, or because discarding it destroys the only complete setup route.

Dragon payloads in Regidrago, dead Battle VIP Pass, Forest Seal Stone after its role is exhausted, Energy, ACE SPECs, and future attackers all demonstrate that DCI changes with board and matchup.

### AMR

Active Move Realism asks whether a theoretically available action is actually executable and sensible now.

Ultra Ball is not "live" merely because it is in hand if the required discards are all critical. A support Pokémon is not automatically available when the Bench is full. A high-cost connector is less attractive if it consumes resources another axis needs.

AMR-overriding occurs when the matchup creates a decisive race: for example, if the opponent will establish a brutal lock next turn, burning normally valuable resources to establish an attacker now can be correct.

### K0 / K1

K0 means the player has not legally inspected enough of the deck to know the Prize cards.

K1 begins once a legal deck search/inspection provides enough information for the player to infer the missing cards by elimination.

Keep engine truth separate from player knowledge. A brain may use Prize knowledge only after it has actually earned K1 in that game.

### ALS

Archetype-line-specifics describe concrete lines a list actually wants to execute, not generic archetype labels.

Examples include:

- Bunnelby + double TM Evolution into Vileplume/Pidgeot;
- Tag Call -> Guzma & Hala -> Thunder Mountain + DCE for an early Iron Thorns attack;
- Horror House -> Electrode self-KOs -> Electro Rain OTK;
- a particular stall deck's recursion + trap loop.

Usage notes exist largely to document these list-specific routes.

### Connector domination

Do not treat access as success.

If Arven -> search Item -> another Pokémon draw engine exists, the brain must still ask what else Arven's Item/Tool access could have done. A connector is dominated when spending it on one path destroys a more complete or more valuable route.

One-shot universal search, ACE SPECs, discard costs, Supporter slots, Bench slots, Energy attachment, evolution timing, and Tools can all be contested connectors.

## Simulation model

A normal trial follows one strong policy trajectory through a complete game.

The simulator is **not** intended to exhaustively solve the full Pokémon TCG game tree. Search targets, discard choices, Bench placement, switch choices, attack targets, hidden information, and opponent responses make exhaustive branching intractable.

Most decisions should use deterministic or state-weighted policy. Expensive reasoning belongs only around choices that can materially change the game.

A useful action loop is:

1. Rules engine generates legal actions.
2. Brain removes strategically dominated options when domination is clear.
3. Brain scores ordinary choices using current objectives.
4. For a small number of critical choices, bounded local projection may compare the strongest candidates.
5. Brain selects one action.
6. Rules engine resolves it.
7. Repeat until the turn/game ends.

Do not recursively branch the full future game from every Trainer, search, discard, or attack target.

### Search and connector realism

The preferred route is the earliest **complete legal route after costs and contention**, not the route with the most theoretical card access.

Supporter-slot contention, discard costs, one-use search, Bench limits, ACE SPEC contention, evolution timing, Energy attachments, and multi-axis cards all matter.

### Hidden information

Public state, each player's private knowledge, and omniscient engine truth must remain distinct.

A production brain receives only the information it is legally entitled to know. Omniscient state may appear in debug traces, but it must never influence the selected action.

### Randomness and reproducibility

All randomness comes from explicit seeded RNG owned by the simulation layer. Avoid hidden random calls inside card or brain modules.

A `GameRecord` must contain enough identity and seed information to reproduce:

- the exact two packages;
- seat assignment;
- opening shuffle;
- Prize cards;
- draws;
- random card effects;
- the resulting game trajectory.

When comparing variants, paired seeds are preferred so the underlying random samples match where possible.

Parallel execution must not change results for a fixed seed set.

## Ending a game

Literal Pokémon TCG victory conditions are always valid terminal states.

Early adjudication is allowed only through a separately tested matchup adjudicator whose predicate has been validated as a real forced or effectively forced state for the exact modeled lists.

Examples include:

- a complete control loop with no live escape route;
- an attacker graph exhausted with no rebuild before the opponent's decisive action;
- an exact combo state that deterministically takes the remaining Prizes.

Favorable position alone is not enough.

Adjudicators live under `src/adjudication/` and return a structured reason rather than a bare boolean. If a shortcut is uncertain, continue playing the game.

A safety turn cap may catch policy loops or unsupported states. Hitting it must be visible as unresolved/error during development and must not silently count as a production win or loss.

## Per-game records

Keep aggregate output small while keeping every game reproducible.

A structured record should be able to capture fields such as:

```text
main_package
opposing_package
seed
main_seat
winner
terminal_type
decisive_turn
decisive_reason
lead_sequence
first_main_attack_turn
first_opponent_attack_turn
locks_established
primary_route
failed_route
recovery_route
policy_notes
```

Full textual traces do not need to be stored for every production trial. They are most useful for sampled audits, failures, unresolved games, policy disagreements, and regression tests.

## Single-game trace mode (`--simulate-this`)

In addition to batch Monte Carlo output, the simulator provides a developer/debug capability equivalent in purpose to the original repository's `--simulate-this` mode:

> Run exactly one complete game between two selected deck packages and show everything needed to audit the game.

The canonical selectors are exact repository-relative **package paths**, not fuzzy archetype names:

```bash
./pokemon_sims_v2 --simulate-this \
  --main "main decks/regidrago-vstar" \
  --opponent "opposing lists/aichi lists/02-takahiro-ando-vileplume-control" \
  --main-goes-first \
  --seed 12345
```

The exact CLI spelling may evolve, but the capability is required. The simulator resolves `decklist.txt` and `brain.cpp` from each selected package.

Seat order must be explicit or clearly reported. A seed is optional for ad hoc runs and accepted explicitly for deterministic replay.

A full trace should report at least:

- exact package paths and resolved decklists/brains;
- RNG seed and seat order;
- mulligans;
- opening hands;
- opening Active/Bench choices;
- Prize setup;
- every draw and random event;
- turn and phase boundaries;
- every action selected by each brain;
- costs, discards, searches, targets, attachments, switches, gusts, evolutions, and attacks;
- relevant candidate-action information for important policy choices;
- policy rationale/scores for materially ambiguous choices;
- bounded projections when used;
- DCI/AMR/route information when it determines the action;
- K0/K1 transitions and the information legally known by the acting player;
- continuous effects and lock changes;
- damage, Special Conditions, Knock Outs, Prize taking, and recovery;
- adjudication checks and terminal reason;
- final board/resources/winner.

For debugging, the trace may also show an explicitly labeled **omniscient engine view** containing hidden cards and Prizes. That view must never be fed back into either brain.

`--simulate-this` uses the **same** rules engine, card implementations, brain code, adjudicators, and RNG semantics as batch production trials. It is not a simplified second simulator.

A suspicious production `GameRecord` should be replayable by feeding its two package paths, seat assignment, and seed into this mode.

## Result generation

For every package under `main decks/`:

1. Recursively enumerate all valid deck packages under `opposing lists/`.
2. Run 1,000 trials with the main package going first against each opponent.
3. Run another 1,000 trials with the main package going second.
4. Aggregate each exact opposing package independently.
5. Produce the vertical matrix and overall row defined by `EXPECTED_RESULT_MANIFEST.md`.
6. Attach deeper diagnostics separately when available.

The matrix should identify opponents by stable repository-relative package path.

`future decks/` is excluded from discovery.

## Testing strategy

Correctness matters more than raw trial count. Tests should identify which layer is wrong.

### Package/discovery tests

Verify:

- package discovery under `main decks/`;
- recursive package discovery under `opposing lists/`;
- `future decks/` exclusion;
- required `decklist.txt`;
- required `brain.cpp`;
- stable package-path identity;
- optional sibling source files do not create phantom decks;
- Usage-only changes do not change parsed card contents.

### Rules tests

Verify objective mechanics with the smallest possible board state: turn rules, evolution timing, Retreat, attack costs, damage, Prize taking, continuous effects, Tools, Stadiums, locks, Special Conditions, and other printed interactions.

### Card tests

Verify exact printings against local card data and authoritative rulings. Prefer reusable effect primitives so a rules correction fixes every card that uses the primitive.

### Brain/policy tests

Give a brain a known legal-information state and verify the strategically correct action or a narrow acceptable set.

Assertions should describe semantic goals rather than overfitting to arbitrary historical turn coordinates or one seed.

### Matchup tests

Exercise real interactions between complete deck packages, including alternate ALS routes, counter-locks, recovery, and escape hatches.

### Regression traces

Fixed seeds are valuable reproducible examples. Tests should assert the strategically meaningful behavior the seed demonstrates rather than treating the seed itself as the specification.

### Reporting tests

Verify seat separation, trial counts, aggregation, combined percentages, recursive package paths, and the matrix contract.

### `--simulate-this` tests

A production `GameRecord` replayed by package paths + seat + seed should reproduce the same game. Trace mode must not change brain decisions.

## Relationship to `pokemon-sims`

The original `pokemon-sims` repository contains substantial Regidrago-specific setup intelligence and regression coverage. Treat it as a behavioral reference while shared helpers and the main Regidrago package brain are implemented.

Useful route logic should be migrated into either:

- shared policy helpers when it is generally reusable; or
- `main decks/regidrago-vstar/brain.cpp` and sibling helpers when it belongs to that exact list.

The old single-player engine structure should not dictate the new two-player state model.

When an inert-opponent state is equivalent to a legacy goldfish scenario, parity tests can compare important decisions and setup timing against the old simulator.

## Performance principles

The primary workload is many short independent games, which parallelizes naturally.

Optimize in this order:

1. correct state transitions;
2. correct card effects;
3. correct brains and adjudication;
4. reproducibility and diagnostics;
5. profiling evidence;
6. targeted optimization of measured hotspots.

Useful optimizations may include compact state objects, immutable card metadata, cached pure continuous-effect queries within a state version, efficient zone containers, batched trials, thread-local RNG, and bounded local projections.

Avoid premature global memoization across hidden-information states unless the state key is proven complete. Incorrect cache equivalence is more damaging than a slower simulation.

The intended workload is policy trajectory simulation, not exhaustive minimax of every legal Pokémon TCG action.

## Agent contribution rules

This repository is expected to receive concurrent agent work. Keep changes easy to merge and audit.

- Read `EXPECTED_RESULT_MANIFEST.md` before implementing simulator output.
- Read the local advanced manual before implementing a game rule.
- Use exact card data before implementing a card effect.
- Consult the Compendium when a rules interaction remains ambiguous.
- Leave `EXPECTED_RESULT_MANIFEST.md` unchanged.
- Leave `future decks/` unchanged unless explicitly tasked.
- Treat `decklist.txt` as exact input data; do not change the 60 cards while implementing engine/brain behavior unless the task is specifically a decklist correction.
- Parse only the PTCGL portion of `decklist.txt`; never count `Usage:` as cards.
- Read Usage before implementing or auditing a deck's brain.
- Put exact-list strategic behavior in that package's `brain.cpp` or sibling helpers.
- Put genuinely reusable strategy in `src/policy/common/` or `src/policy/archetypes/`.
- Keep generic rules free of archetype strategy.
- Keep policy logic out of card-effect handlers.
- Put matchup terminal shortcuts in adjudication modules and test them independently.
- Prefer shared primitives over repeated card-specific rule implementations.
- Avoid giant archetype switches and one-off seed fixes.
- Add direct source URLs beside rules-sensitive implementation code.
- Keep commits focused on coherent subsystems.
- Add tests with each mechanic, card family, brain rule, or adjudication predicate.
- Inspect current branches/recent commits before editing shared files so concurrent work is not overwritten.
- Prefer adding isolated modules behind stable interfaces over editing one giant central file.

## Design goal

The simulator makes the combinatorics manageable by separating four questions:

1. **What exact deck is this?** `decklist.txt` answers this.
2. **What actions are legal and what do they do?** The rules/card layers answer this.
3. **Which legal action should this exact deck choose?** Its `brain.cpp`, supported by shared policy helpers, answers this.
4. **Has the game actually ended or reached a validated forced state?** Literal rules and adjudication answer this.

That separation allows independent agents to improve deck strategy without corrupting rules, improve rules without rewriting every deck, and replay any suspicious simulated game through the same exact package code that produced it.
