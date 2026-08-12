# Pokémon Sims v2

A two-player Pokémon TCG **paper Expanded** simulator intended to play complete games between selected main decks and real opposing decklists, then measure matchup win rates by seat order.

The project is deliberately organized around a small rules core, deck-specific strategy modules, exact decklist inputs, reproducible Monte Carlo trials, and strict reporting contracts. The problem has a large combinatorial search space, so architecture should make correctness easy to audit while keeping expensive reasoning localized.

## Authoritative project contracts

Read these before changing simulator behavior:

1. [`EXPECTED_RESULT_MANIFEST.md`](EXPECTED_RESULT_MANIFEST.md) defines the required end result and output schema. **Do not edit it.**
2. [`official documentation/EN_advanced_manual-2025-transcription-structured.md`](official%20documentation/EN_advanced_manual-2025-transcription-structured.md) is the first rules reference for implemented game mechanics.
3. [`official documentation/compendium-ruling-guidance`](official%20documentation/compendium-ruling-guidance) explains when to use the Pokémon TCG Compendium for interaction-specific rulings.
4. For agents: your local environment should have: `tcg-data-master`, this is the card-data source for exact card text and printing data during development. Card behavior should be tied to exact printings rather than inferred from a card name.

Rules-based code should carry a direct authoritative rules, ruling, or card-data URL beside the relevant implementation when practical. Strategic policy and rules legality must stay separate.

## Required result

For every deck file in `main decks/`, recursively discover every deck file under `opposing lists/` and simulate the complete matchup.

Each main-deck/opposing-list pairing runs:

- 1,000 games with the main deck going first
- 1,000 games with the main deck going second

The required primary report is one vertical matchup matrix per main deck containing the first-seat win rate, second-seat win rate, combined win rate, and trial count for every opposing list. The exact schema and examples live in [`EXPECTED_RESULT_MANIFEST.md`](EXPECTED_RESULT_MANIFEST.md).

Every trial represents one complete game. Item lock, Ability lock, Supporter lock, lead Pokémon, setup races, disruption, recovery, alternate routes, prizing, and matchup-specific interactions should arise from the real simulated board. Separate synthetic lock-only simulations are outside the primary result contract.

Detailed diagnostics are encouraged. Useful fields include decisive turn, win condition, lead sequence, lock establishment, failed route, recovery route, reason for loss, major policy decisions, and adjudication reason. These fields supplement the required win-rate matrix.

## Repository inputs

```text
pokemon-sims-v2/
├── EXPECTED_RESULT_MANIFEST.md
├── README.md
├── main decks/
│   └── regidrago-vstar.txt
├── opposing lists/
│   └── aichi lists/
│       └── ...
├── future decks/
│   └── README
└── official documentation/
    ├── EN_advanced_manual-2025-transcription-structured.md
    └── compendium-ruling-guidance
```

### `main decks/`

Active decks whose matchup matrices are being measured. A newly added file here automatically becomes another main-deck simulation target.

### `opposing lists/`

Every deck file anywhere below this directory is an independent opponent. Subdirectories are organizational. Tournament provenance, archetype, player, and exact list may differ between files.

Do not merge similar lists into one abstract opponent. Small list differences can change opening lines, recovery, lock timing, and matchup outcomes.

### `future decks/`

Quarantined inputs. Leave this directory alone unless the project owner explicitly moves a deck into the active simulation set. Its local README is authoritative for that folder.

### `official documentation/`

Local rules references. Use the advanced manual first. Consult the Compendium when the manual does not resolve a specific interaction cleanly.

## Decklist format and `Usage:` addendum

Deck files in `main decks/`, `opposing lists/`, and eventually promoted decks use the **Pokémon TCG Live deck import/export text format** as the machine-readable decklist portion. The simulator should accept the same basic structure produced by PTCGL: section headers such as `Pokémon:`, `Trainer:`, and `Energy:`, followed by quantity + card name + set code + collector number lines, and ending with `Total Cards: 60`.

A normal deck file therefore looks like:

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

The repository extends that format with one optional, human-readable strategy appendix. Exactly two newline characters after the completed `Total Cards: 60` line may begin:

```text
Usage:
<text>
```

In raw form, the boundary is:

```text
Total Cards: 60\n\nUsage:\n...
```

Everything from the `Usage:` marker through end-of-file is **commentary, not deck data**. The deck parser must stop processing card-list data at that boundary and must not interpret, validate as cards, count, or execute anything in the Usage body. Any amount of text is allowed after `Usage:`, including multiple paragraphs, blank lines, punctuation, card names, example lines that resemble PTCGL entries, matchup notes, or other free-form prose.

`Usage:` is optional. A normal PTCGL import that ends at `Total Cards: 60` remains valid and represents the same deck as an otherwise identical file with a Usage appendix. The appendix never changes the 60-card contents.

The purpose of `Usage:` is to give agents and policy authors high-level context about how the exact list intends to play. It can describe an archetype's normal game plan, ALS/opening sequences, control objectives, lead Pokémon priorities, important recovery loops, matchup-specific goals, or traps that a generic card-by-card evaluator could misunderstand. For example, a Snorlax Stall list should explain that its primary plan is to trap and exhaust the opponent rather than behave like an attacking deck; piloting the same 60 cards under the wrong strategic objective can make an otherwise accurate rules simulation meaningless.

A Usage appendix is **guidance, not rules text and not a hardcoded move script**. Agents should use it to understand the deck and then implement or improve the appropriate policy module. The runtime deck parser should ignore it. If future tooling exposes Usage text to development tools, it must remain separate from the parsed card list and must not silently override game rules, hidden information, or legal-action generation.

Example:

```text
Pokémon: 11
3 Snorlax PGO 55
...

Trainer: 44
4 Plumeria BUS 120
...

Energy: 5
3 Capture Energy RCL 171
2 Water Energy MEE 3

Total Cards: 60

Usage:
This is a stall/control deck. Its primary objective is to strand an opposing Pokémon
that cannot attack effectively, maintain Block/retrap pressure, recur Supporters, and
remove Energy or escape resources. Do not evaluate the deck as though its main goal
were to race for damage with Snorlax.

Additional lines and matchup notes are allowed here. None of this text is part of the
60-card decklist.
```

Parser tests should explicitly verify that:

- a stock PTCGL export ending at `Total Cards: 60` parses normally;
- the same export plus `\n\nUsage:\n...` produces exactly the same 60 parsed cards;
- arbitrary multiline Usage text is ignored by the deck parser;
- card-looking lines inside Usage do not add cards or alter counts;
- malformed deck data before `Total Cards: 60` is not excused by a Usage appendix;
- deck identity, hashing, and simulation pairing should be based on the parsed cardlist/path as designed, not accidentally changed by commentary text unless a separate explicit metadata hash is desired for tooling.

## Target source architecture

The source tree should grow toward the following boundaries. These directories are architectural targets, so create them only as implementation work reaches them.

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
│   ├── regidrago/
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
├── matchups/
└── reporting/
```

The exact filenames may evolve. The module boundaries are more important than the spelling of a directory.

## Architectural rules

### The engine owns game mechanics

The rules layer should know how a legal Pokémon TCG game changes state. It owns turn sequencing, legal-action generation, costs, zones, evolution timing, Energy attachment, Retreat, attacks, damage, Knock Outs, Prize taking, Special Conditions, continuous effects, and other objective rules.

The engine should not know that Regidrago wants a Dragon payload in the discard pile or that a Vileplume list wants a Bunnelby + TM Evolution opening. Those are strategic facts and belong in policy modules.

Avoid generic engine branches keyed to archetype names. If a board interaction can be expressed through actual card effects and current state, model the interaction there.

### Card implementations own card effects

A card module translates the printed card into game effects. It should not decide whether playing that card is strategically desirable.

Card identity must preserve exact printing information. Deck files already provide set and collector-number information, which should resolve through the card-data adapter to a stable internal `CardId`.

Shared effects should be reusable. A generic switch, search, gust, draw, discard, damage-modifier, Ability-suppression, attack-cost, Item-lock, or Supporter-lock primitive should not be reimplemented for each archetype.

### Policies own strategy

A policy receives the information legally available to that player and selects among legal actions.

Policy is where deck intelligence belongs, including:

- archetype-line-specifics and opening plans
- dynamic Discard Capable Index decisions
- Active Move Realism
- K0/K1 deck and Prize knowledge
- Supporter contention
- connector domination
- resource preservation
- lead Pokémon selection and ordered lead sequences
- matchup-aware target selection
- recovery planning
- local tactical projection when an important choice is genuinely ambiguous

A policy may know the opponent's publicly revealed archetype and board. It must not read hidden opposing cards or unknown Prize information.

An exact tournament list may need strategy that differs from another list in the same archetype. Keep reusable archetype policy in shared modules, with narrow list-specific behavior only when the exact list creates a real ALS or interaction difference.

### Lead Pokémon are a strategic role

Do not encode `LeadPokemon` as a special rules category. Goomy, Wobbuffet, Budew, Klefki, Iron Thorns ex, and similar cards are ordinary Pokémon whose early-game value is recognized by policy.

Their effects should operate through the normal board and rules system. Ordered plans such as Wobbuffet into Goomy must remain distinguishable from the reverse order because the resulting setup graphs can differ radically.

### Locks emerge from the board

Do not represent a real matchup as a synthetic `item_lock = true` scenario. Vileplume should create Item lock because the correct Vileplume is in play with an active Ability. Wobbuffet should suppress the relevant Abilities because it is Active and its effect applies. Goomy should tax attacks through its printed effect.

This keeps interacting locks composable and allows switching, gusting, Tool removal, Ability suppression, Stadiums, Knock Outs, and other escape hatches to work naturally.

## Simulation model

A normal trial should follow one strong policy trajectory through a complete game.

The simulator is not intended to exhaustively solve the full Pokémon TCG game tree. That approach becomes combinatorially intractable once searches, discard choices, Bench choices, targets, switch decisions, and opponent responses multiply across turns.

Most decisions should use deterministic or state-weighted policy. Expensive reasoning belongs only around decisions where the choice can materially change the game.

A useful decision flow is:

1. Generate legal actions from the rules engine.
2. Remove strategically dominated choices when policy can prove domination.
3. Score ordinary choices from current state and deck objectives.
4. For a small number of critical choices, project the strongest candidates through a bounded local horizon.
5. Execute one action and continue the real game.

Do not recursively branch the entire game from every Trainer, search target, discard, or attack target.

### Search and connector realism

Access to a card does not prove a route is good. Policies must account for the cost of the connector and what that same connector could have accomplished elsewhere.

Examples include Supporter-slot contention, one-use universal search, discard costs under low DCI, occupied Bench slots, ACE SPEC contention, and multi-axis cards that satisfy several setup requirements at once.

The simulator should prefer the earliest complete legal route after all costs and contention are applied. A route that reaches a card but destroys a more important axis can be dominated.

### Hidden information

Keep public game state separate from each player's private knowledge.

K0 means the player has not legally inspected the deck and therefore cannot infer the Prize cards. K1 begins after a legal full-deck search or inspection gives enough information to determine the missing cards by elimination.

Policies may use K1 information only after it has actually been earned in that trial.

### Randomness and reproducibility

All randomness should come from explicit seeded RNG owned by the simulation layer. Avoid hidden random calls inside card or policy modules.

A game record should contain enough seed information to reproduce the exact opening, Prizes, draws, and random effects. When comparing policy or deck variants, paired seeds are preferred so differences are measured against the same underlying random samples where possible.

Parallel execution must not change results for a fixed seed set.

## Ending a game

Literal Pokémon TCG victory conditions are always valid terminal states.

Early adjudication is allowed only through a separately tested matchup adjudicator whose predicate has been validated as a real checkmate or effectively forced state for the exact modeled lists. Favorable board position alone is not sufficient.

Examples of adjudication candidates include a fully established control loop with no live escape route, an exhausted attacker graph that cannot rebuild before the opponent's decisive action, or an exact combo state that deterministically takes the remaining Prizes.

Adjudicators belong under `src/adjudication/`. They should return a structured reason rather than a bare boolean. If a proposed shortcut is uncertain, continue playing the game.

A safety turn cap may exist to catch policy loops or unsupported states. Hitting that cap should be visible as an unresolved/error result during development and should not be silently counted as a win or loss in the final matrix.

## Per-game records

Keep the required aggregate output small while making individual trials auditable.

A structured game record should be able to capture fields such as:

```text
main_deck
opposing_list
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

Avoid storing enormous text traces for every production trial. Full traces are most useful for sampled audits, failures, unresolved games, policy disagreements, and regression tests.

## Single-game trace mode (`--simulate-this`)

In addition to batch Monte Carlo output, the simulator should provide a developer/debug interface equivalent in purpose to the original repository's `--simulate-this` mode: **run exactly one complete game between two selected deck files and print enough information to audit every important decision and state transition.**

The canonical selector should be the exact repository-relative deck path so similarly named archetypes never become ambiguous. A target CLI may look like:

```bash
./pokemon_sims_v2 --simulate-this \
  --main "main decks/regidrago-vstar.txt" \
  --opponent "opposing lists/aichi lists/02-takahiro-ando-vileplume-control.txt" \
  --main-goes-first \
  --seed 12345
```

The exact flag spelling may evolve, but the capability is required. It should be possible to ask, in effect, **"run one game and show me everything for Regidrago VSTAR versus the Vileplume control list in Aichi slot #2."** Seat order must be explicit or clearly reported. A seed should be optional for ad hoc runs and accepted explicitly for deterministic replay.

The trace should make the game inspectable without changing how either policy plays. At minimum, it should report:

- exact main and opposing deck files
- RNG seed and seat order
- mulligans, opening hands, opening Active/Bench choices, and Prize setup
- every draw and other random event
- turn and phase boundaries
- every action selected by each policy
- costs paid, cards discarded, cards searched, targets chosen, attachments, switches, gusts, evolutions, and attacks
- relevant legal-action or candidate-action information when it helps explain a policy decision
- policy rationale or scores for materially ambiguous choices, including bounded projections when used
- DCI/AMR or route information when those values materially determine the action
- K0/K1 transitions and what the acting policy is legally allowed to know
- continuous effects and lock changes caused by the actual board
- damage, Special Conditions, Knock Outs, Prize taking, recovery, and zone changes
- adjudication checks and the exact terminal/decisive reason
- final board, remaining resources, winner, and termination type

For debugging, the trace may also provide an explicitly labeled **omniscient engine view** containing hidden cards and Prize contents. That view must never be fed into policy decisions. The trace should distinguish engine truth from each player's legal information so hidden-information bugs are easy to detect.

`--simulate-this` should use the same rules engine, card implementations, policies, adjudicators, and RNG semantics as production trials. It must not be a second simplified simulator. A production `GameRecord` should contain enough identifiers that a suspicious game can be replayed through this mode using the same two deck paths, seat assignment, and seed.

This mode is a primary correctness tool for agents. New mechanics, policies, matchup adjudicators, and surprising statistical results should be auditable by running representative fixed seeds and reading the complete trace from opening setup through termination.

## Result generation

The reporting layer must follow [`EXPECTED_RESULT_MANIFEST.md`](EXPECTED_RESULT_MANIFEST.md).

For every main deck:

1. Recursively enumerate all files under `opposing lists/`.
2. Run 1,000 trials with the main deck going first for each opposing list.
3. Run another 1,000 trials with the main deck going second.
4. Aggregate each exact opposing list independently.
5. Produce the vertical matrix and overall row defined by the manifest.
6. Attach deeper statistics separately when available.

`future decks/` is excluded from discovery.

## Testing strategy

Correctness matters more than raw trial count. Tests should be layered so failures identify the responsible module.

### Rules tests

Verify objective mechanics with the smallest possible board state. Evolution timing, Retreat, attack costs, Prize taking, continuous effects, lock interactions, Tool behavior, and similar rules belong here.

### Card tests

Verify each implemented card against its exact printed text and authoritative ruling sources. Prefer reusable effect primitives so a rules correction fixes every card that uses the primitive.

### Policy tests

Give a policy a known legal state and verify the strategically correct action or a narrow acceptable action set. Assertions should describe semantic goals instead of overfitting to historical turn numbers or arbitrary seed coordinates.

### Matchup tests

Exercise real interactions between policies, including escape hatches and alternate ALS routes. A lock test should verify how the opposing policy responds when the primary route is disconnected.

### Regression traces

Fixed seeds are useful as reproducible examples. The test should assert the strategically meaningful behavior demonstrated by the seed rather than making the seed itself the specification.

### Reporting tests

Verify deck discovery, seat separation, trial counts, aggregation, combined percentages, recursive opponent paths, `future decks/` exclusion, and the exact matrix contract.

## Relationship to `pokemon-sims`

The original `pokemon-sims` repository contains substantial Regidrago-specific setup intelligence and regression coverage. Treat it as a behavioral reference while Regidrago policy is migrated.

Useful strategic concepts and validated route logic should be ported into `src/policy/regidrago/` or reusable policy helpers. The old single-player engine structure should not dictate the new two-player state model.

When an inert-opponent state is equivalent to a legacy goldfish scenario, parity tests can compare important decisions and setup timing against the old simulator. This protects accumulated Regidrago intelligence while allowing the v2 rules engine to remain general.

## Performance principles

The primary workload is many short independent games, which parallelizes naturally.

Optimize in this order:

1. Correct state transitions.
2. Correct policies and adjudication.
3. Reproducibility and diagnostics.
4. Profiling evidence.
5. Targeted optimization of measured hotspots.

Useful optimizations may include compact state objects, immutable card metadata, cached pure continuous-effect queries within a state version, efficient zone containers, batched trials, thread-local RNG, and bounded local projections.

Avoid premature global memoization across hidden-information states unless the state key is proven complete. Incorrect cache equivalence is more damaging than a slower simulation.

## Agent contribution rules

This repository is expected to receive concurrent agent work. Keep changes easy to merge and easy to audit.

- Read `EXPECTED_RESULT_MANIFEST.md` before implementing simulator output.
- Read the local advanced manual before implementing a game rule.
- Use exact card data before implementing a card effect.
- Consult the Compendium when a rules interaction remains ambiguous.
- Leave `EXPECTED_RESULT_MANIFEST.md` unchanged.
- Leave `future decks/` unchanged unless explicitly tasked.
- Treat decklist files as input data. Avoid changing them while implementing engine or policy behavior unless the task is specifically a decklist correction or Usage documentation task.
- Parse decklists as PTCGL import text with the optional `Usage:` appendix described above; never count Usage text as cards.
- Read a deck's `Usage:` notes when developing its policy so the simulator does not optimize toward the wrong archetype objective.
- Keep generic rules free of archetype strategy.
- Keep policy logic out of card effect handlers.
- Put matchup terminal shortcuts in adjudication modules and test them independently.
- Prefer shared primitives over repeated card-specific implementations.
- Avoid giant archetype switches and one-off seed fixes.
- Add direct source URLs beside rules-sensitive implementation code.
- Keep commits and pull requests focused on one coherent subsystem when practical.
- Add tests with each new mechanic, card family, policy rule, or adjudication predicate.
- Inspect current branches and recent commits before editing shared files so concurrent work is not overwritten.

When several agents need to work at once, prefer adding isolated modules behind stable interfaces over editing one central file.

## Design goal

The simulator should make the combinatorics manageable by separating three questions:

1. **What actions are legal?** The rules and card-effect layers answer this.
2. **Which legal action should this deck choose?** The policy layer answers this using matchup-aware strategy and bounded projection.
3. **Has the game actually ended or reached a validated forced state?** Literal rules and adjudication answer this.

That separation allows the simulator to play complete paper Expanded games with realistic locks, setup races, lead Pokémon, recovery, and disruption while keeping each subsystem understandable enough for independent agents to improve safely.
