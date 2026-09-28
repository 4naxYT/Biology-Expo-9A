<!-- Project documentation updated with AI assistance. -->
# Bio-Expo-9A

> A portable, self-contained artificial-life simulation in which digital
> organisms evolve under selection pressure inside a silicon petri dish.

This is a Windows desktop application that grows, mutates, predates,
cooperates, and extincts a population of autonomous digital organisms. Each
organism ("specimen") is a small neural network whose weights are inherited
from its parent, mutated with each generation, and shaped over hundreds of
generations by an unforgiving energy economy. The result is a real-time
demonstration of Darwin's theory of natural selection playing out on your
screen, rendered as a live arena, a speciation chart, a fitness timeline, and
a directory of persistent learning artifacts.

The project is deliberately portable: copy the whole folder to a USB stick,
run `Install Deps.bat` once on a machine with Python, and it will build its
own virtual environment and launch on any Windows machine with WebView2
(bundled with Windows 10+).

---

## Table of contents

1. [Quick start](#quick-start)
2. [What you're looking at](#what-youre-looking-at)
3. [The simulation model](#the-simulation-model)
4. [The organism](#the-organism)
5. [The brain](#the-brain)
6. [Evolution and inheritance](#evolution-and-inheritance)
7. [Ecology: interactions between organisms](#ecology-interactions-between-organisms)
8. [Environmental pressure](#environmental-pressure)
9. [Relation to real biology](#relation-to-real-biology)
10. [How this demonstrates Darwin's theory](#how-this-demonstrates-darwins-theory)
11. [Architecture](#architecture)
12. [Directory layout](#directory-layout)
13. [Configuration reference](#configuration-reference)
14. [Controls and commands](#controls-and-commands)
15. [Python concepts used in this codebase](#python-concepts-used-in-this-codebase)
16. [Development](#development)
17. [AI assistance](#ai-assistance)

---

## Quick start

1. Run `Install Deps.bat`. This creates `runtime/venv`, installs PyTorch (CPU
   build), `aiohttp`, `numpy`, and `pywebview`, then exits.
2. Run `Run.bat`. Two windows open: the **Visualiser** (the arena) and the
   **Dashboard** (statistics, controls, and the console).
3. Press `Space` to pause, `A` to drop an asteroid, `S` to save the current
   population to a slot. See the [controls](#controls-and-commands) section
   for the full list.

The backend binds to `127.0.0.1` on an auto-selected free port. Everything is
local; no network traffic leaves your machine.

---

## What you're looking at

The **Visualiser** is the arena. Coloured Arrows are individual specimens; the
hue of each Arrow encodes its heritable colour gene, which acts as a proxy for
species identity. Food is a green Dot, poison is a magenta Dot, rogues are white, and the
background darkens through the day/night cycle. Right-click to attract every
creature toward a point. Left-click to drop resources. Click a creature to
inspect its brain.

The **Dashboard** has three tabs:

- **Live Info** — species population plot (Muller-style stacked area chart),
  dominant-species trend analysis, event log, live simulation console, god-mode
  resource drops, save/load slots, and the neural stack.
- **Customisability** — live-tunable parameter sliders (vision range, energy
  values, mutation rates, spawn rates, etc.) that write back into the running
  simulation. Values persist to `config.json`.
- **Nural View** — inspection of the persisted `Nural/` artifact tree, with
  a node-link visualisation of saved brains, a weight heatmap of the currently
  best genome, activation monitor for the inspected creature, and a fitness
  timeline across generations.

---

## The simulation model

This project advances in discrete ticks. At 60 Hz (`target_tick_hz`), each tick
does the following, in order:

1. **Item decay.** Every active food or poison item ages; once it exceeds
   `food_decay_ticks` or `poison_decay_ticks`, it disappears. Poison loses
   potency gradually (`poison_potency_decay`) until a floor of 15%.
2. **Immigration.** A tiny trickle of brand-new, randomly-hued creatures
   enters the arena from the left or right edge with probability
   `immigration_per_tick` per tick. This prevents total extinction from being
   a permanent absorbing state.
3. **Renewable food.** With probability `food_spawn_per_tick` per tick, one
   new item spawns at a uniformly random location. It is poison with
   probability `poison_spawn_share`, otherwise food.
4. **Radiation toxic rain.** If `env_radiation` exceeds
   `radiation_rain_threshold`, extra poison items fall in proportion to the
   radiation level.
5. **Per-creature update.** Movement, feeding, metabolism, idle drain, energy
   decay, and death by starvation or external cause.
6. **Collisions.** All creature pairs within 10 units undergo an interaction:
   anti-clump pushback, rogue combat, horizontal gene transfer, kin sharing,
   or predation — depending on the pair's relationship.
7. **Mitosis.** Any creature whose energy exceeds `reproduce_threshold`
   spawns one child.
8. **Extinction check.** If the living population reaches zero, an
   `Extinction` event fires and the arena is reseeded from the trainer's
   Hall of Fame. The generation counter increments.

The cycle is capped by `max_creatures` (default 50) and item slots (240).

---

## The organism

A `Creature` is a lightweight data object with the following heritable and
non-heritable state:

| Field | Meaning | Heritable? |
|---|---|---|
| `x`, `y`, `angle` | Position and facing | No (body state) |
| `hue` | Colour gene in [0, 360) | Yes (mutates ±10° on birth) |
| `size` | Body size, currently near-constant | Yes (95% of parent) |
| `energy` | Metabolic currency | No |
| `is_rogue` | Whether the creature is a predator morph | Yes (rare spawn) |
| `id_str` | Unique `S-#####` tag | No |

Every tick a creature pays a **metabolic cost**:

```
metabolism = (base_metabolism + |speed| × speed_metabolism) × (1 + radiation × 0.03)
```

This is directly modelled on **Kleiber's law**-style scaling: there is a fixed
cost to existing (basal metabolic rate) and a variable cost proportional to
activity. Moving is expensive. Sitting still is cheaper but triggers an idle
penalty that drains fitness and energy over time, punishing the strategy of
"do nothing and hope kin share food with you".

Energy is the sole currency. Eating food adds `food_energy`. Eating poison
subtracts `poison_damage × potency`. Reaching `reproduce_threshold` splits the
creature in two. Falling to zero kills it, and the corpse decays into a poison
item — a small but real nod to nutrient recycling.

---

## The brain

Each creature's behaviour is driven by a small neural network implemented in
PyTorch (CPU-only). The default architecture is:

```
42 inputs → 2 hidden layers × 64 units → 2 outputs
```

with `tanh` or ReLU activations between layers (see `Brains/network.py`).
The two outputs are the creature's **speed** and **turn** commands, both in
`[-1, 1]` after passing through `tanh`.

### The 42 sensory inputs

The input vector is deliberately large because the creature needs a
surrounding world to react to, not just a single "nearest item" signal.
Eight-way directional cones give the creature a coarse, wasp-eye view:

| Index range | Meaning |
|---|---|
| 0–7 | Nearest food distance in each of 8 cones (1.0 = out of range) |
| 8–15 | Nearest poison distance per cone |
| 16–23 | Nearest **kin** (same hue within `kin_threshold_hue`) per cone |
| 24–31 | Nearest **rogue** (predator) per cone |
| 32–35 | Wall proximity: N, E, S, W |
| 36 | Own energy, normalised |
| 37 | Own age, normalised |
| 38 | Day/night phase |
| 39 | Recent food eaten (decays each tick) |
| 40 | Recent damage taken (decays each tick) |
| 41 | Count of kin in vision radius, normalised |

The cone representation is important. It means the brain can represent
*direction* to targets, not just their existence. This is what makes
phototaxis-like navigation around obstacles and predators possible.

### Why a real network instead of a hand-written controller

A hand-written controller would pre-suppose we already know what a good
foraging strategy looks like. A neural network does not. It lets evolution
discover non-obvious strategies — grazing patterns, wall-skimming,
predator evasion, kin clustering — that a human designer would not have coded.

The trade-off is that we now need selection to actually select. The next
section is about how.

---

## Evolution and inheritance

This project uses **neuroevolution**, not gradient descent. There is no
backprop, no labelled training set, no loss function in the machine-learning
sense. Instead, the population's fitness is measured by the world and the
best genomes are kept, mutated, and re-seeded.

### Selection

At the end of each generation (whenever the population goes extinct, or on
a manual rollover), each creature's `fitness` is computed as a weighted sum
of its per-generation achievements:

```
fitness = w_food × food_eaten
        + w_lifespan × ticks_lived
        + w_reproductions × offspring_count
        + w_kills × successful_kills
```

with the weights (`fitness_w_*`) configurable per **goal**. The four
built-in goals are:

- **Most_Food** — emphasises `food_eaten`
- **Longest_Lifespan** — emphasises `ticks_lived`
- **Most_Reproductions** — emphasises offspring
- **Kills** — emphasises rogue kills

Switching goals changes which behaviours are rewarded, and you can watch
populations diverge accordingly. This is a direct demonstration of how
*selection pressure* defines the direction of evolutionary change.

### Mutation

Every child's genome is perturbed from its parent's. Two mutation regimes
operate in parallel, both configurable:

- **Point mutation** (`mutation_base`, `mutation_severity`): each weight is
  nudged by a small amount drawn from a Gaussian or uniform distribution.
  This is the classic Darwinian "small, undirected variation" mechanism.
- **Saltation** (`saltation_rate`): with low probability, a weight is
  replaced entirely with a fresh random value. This models large-effect
  mutations that occasionally produce dramatic phenotypic jumps.

Saltation is the digital equivalent of the kind of mutation that gives a
bacterium a completely new metabolic pathway in a single generation. Real
biology rarely does this, but it happens — and it is important for escaping
local fitness minima.

### Crossover

Two sexually compatible creatures (same hue within `kin_threshold_hue`, both
above `crossover_min_energy`) can, with probability `crossover_rate`, produce
a child whose weight vector is a **recombination** of both parents. This is
meiosis in miniature. Cross-hue crossover is much rarer
(`crossover_rate_cross_hue`) and is punished by a `cross_hue_mutation_mult`
multiplier on the offspring's mutation rate — a nod to the reduced viability
of distant hybridisation in real biology.

### Horizontal gene transfer

Creatures of any relationship can, on contact, with probability
`hgt_chance` per tick, exchange a single random weight index. This is not
inheritance — it's **horizontal gene transfer**, the mechanism by which
bacteria share antibiotic resistance genes across species boundaries. In
this project it lets a successful weight "leak" across a population even
between creatures that cannot sexually reproduce with each other, which is
exactly what happens in real bacterial populations.

### Speciation

Every child inherits its parent's hue, plus a small random jitter (±10°).
Over generations, hue drifts. When two lineages drift far enough apart in
hue that `hue_diff > kin_threshold_hue`, they stop recognising each other
as kin — and immediately start predating each other instead. This is a
digital **speciation event**: two formerly cooperating lineages have become
reproductively isolated, and now compete for resources.

You can watch this on the Muller plot, where species bands appear, split,
drift, and vanish.

---

## Ecology: interactions between organisms

When two creatures come within 10 units, one of four things happens:

1. **Anti-clump pushback.** They push apart. This is a physical crowding
   term, not a biological one — it prevents the whole population from
   collapsing into a single point.

2. **Rogue combat.** If one is a rogue and the other is not, the rogue deals
   `rogue_damage_per_tick` damage and takes `rogue_counter_damage` in return.
   A kill grants the rogue `rogue_kill_heal` energy. Rogues heal from poison
   (`rogue_poison_heal`) and lose energy from food (`rogue_food_damage`) —
   they are obligate carnivores.

3. **Kin sharing.** If both are within `kin_threshold_hue` of each other,
   they average their energies. This is mutualism — a soft form of
   **kin selection** in the Hamilton's-rule sense. The shared hue, which is
   heritable, is a marker of relatedness, and mutual aid flows toward
   relatives.

4. **Predation.** If they are not kin, the higher-energy creature steals
   `predation_steal` from the lower. This is straightforward competition.

HGT and crossover requests are also queued during collisions and resolved by
the trainer at the end of the tick.

---

## Environmental pressure

### Day/night cycle

`day_cycle_ms` controls the length of a full day in wall-clock milliseconds.
Vision range scales from 50 to 150 units across the cycle. Night is a real
hazard: creatures cannot see distant food or predators, so different
strategies win at different times of day. This maps onto the real behaviour
of nocturnal versus diurnal foragers.

### Radiation

`env_radiation` is a scalar [0, 100] that can be driven by:

- the **WiFi scanner** (`Core/radiation.py`), which counts nearby access
  points and their signal strengths as a proxy for environmental RF
  pollution;
- manual input from the dashboard sliders;
- any external driver that writes to the WebSocket.

Radiation has three effects:
- it speeds up metabolism,
- it introduces random positional jitter above a threshold,
- above `radiation_rain_threshold`, it causes poison to precipitate out of
  the air.

This is a real, physical environmental stressor. In real biology, radiation
drives mutation and DNA damage; here it drives metabolic penalty and
environmental poison, both of which increase selection pressure and therefore
speed up evolution.

### Density dependence

The population is capped at `max_creatures`. When the arena is full,
mitosis fails and the parent is throttled. This is a hard **carrying
capacity**, and its effects on the population are the same as in real
ecosystems: competition intensifies, more mutations are culled, and the
average phenotype shifts toward efficiency.

---

## Relation to real biology

This project is not a biological model. It is a **toy model** that abstracts
real biology down to a tractable computational core, but the abstractions
are chosen to preserve the mechanisms that Darwin identified as necessary
for natural selection to operate.

| Real biology | Bio-Expo-9A analogue |
|---|---|
| Genome | Weight vector of a neural network + hue + size |
| Phenotype | Observed behaviour and colour |
| Point mutation | Small Gaussian jitter on a weight |
| Large-effect mutation | Saltation: weight replaced entirely |
| Recombination (meiosis) | Crossover between kin above an energy threshold |
| Bacterial conjugation | Horizontal gene transfer on collision |
| Basal metabolic rate | `base_metabolism` per tick |
| Activity cost | `speed_metabolism × |speed|` |
| Foraging | Eating food items for energy |
| Toxin avoidance | Poison damage |
| Predation | Rogue contact damage and energy theft |
| Kin recognition | Hue similarity below `kin_threshold_hue` |
| Speciation | Hue divergence beyond `kin_threshold_hue` |
| Carrying capacity | `max_creatures` cap |
| Reproductive isolation | Cross-hue crossover penalty |
| Nutrient cycling | Dead creatures become poison items |
| Environmental mutagen | Radiation |
| Circadian rhythm | Day/night vision cycle |

Some of these analogies are strong (the metabolic cost is essentially a
simplified Kleiber's law). Some are looser (the neural network is a
considerable simplification of a nervous system). None of them are intended
as *predictive* biology. They are intended as *pedagogical* biology: a way to
watch selection operate on mechanisms the way it operates on flesh and blood.

Notable simplifications:

- **No genome-phenotype separation.** The weights *are* the behaviour. Real
  organisms have layers of gene regulation, protein folding, and development
  between genotype and phenotype. Skipping this speeds up evolution but
  removes the neutral drift and canalisation that shape real genomes.
- **No diploidy.** Everything is haploid. This makes crossover simpler, but
  it eliminates recessive alleles and the hiding of genetic variance that
  diploidy provides.
- **No ageing.** Creatures die from starvation, combat, or poisoning — never
  from old age. Lifespan is a fitness weight, not a fixed maximum.
- **No niches.** Every creature lives in the same 800×800 arena with the same
  resource distribution. Real ecology is spatially structured; this project
  is not.

These are intentional design choices. They make the simulation fast enough
to run at 60 Hz on a laptop, and — more importantly — make the causal chain
from "gene" to "selection" short enough for a human being to follow.

---

## How this demonstrates Darwin's theory

Darwin's *On the Origin of Species* can be reduced to four postulates, plus
a conclusion. This project implements all four explicitly and lets you watch
the conclusion fall out.

### 1. Variation

*Individuals within a population vary in their traits.*

Every creature's neural weights are randomly initialised at seeding time,
then independently mutated at each reproduction event. Two children of the
same parent, over several generations, will diverge. At any moment, the
population contains a spread of strategies — some better at foraging, some
better at fleeing, some that sit still and die, some that patrol the walls.

**You can see this** on the Visualiser immediately after a reset: fifteen
creatures, all with the same starting state, immediately diverge into fifteen
different movement patterns.

### 2. Heritability

*Offspring resemble their parents more than they resemble unrelated
individuals.*

Weights are copied from parent to child (with small perturbations), and hue
is inherited with ±10° jitter. Since hue determines kin recognition and
therefore kin sharing and mate choice, relatedness is explicitly tracked and
explicitly advantageous.

**You can see this** on the Muller plot as species bands that persist across
generations — a lineage is a visible, heritable, coloured thread in time.

### 3. Differential survival and reproduction

*Some variants survive and reproduce better than others in a given
environment.*

Energy is the currency. A creature that finds food efficiently gains energy
and reproduces; one that doesn't, starves. A creature that walks into a
poison cloud loses energy and may die. A creature that gets caught by a rogue
loses energy and may die. All of these are non-random with respect to
phenotype: they select for whatever neural patterns produce successful
foraging and evasion.

**You can see this** in the fitness timeline on the Nural tab: mean fitness
rises over generations as selection prunes unsuccessful weights.

### 4. Descent with modification

*Over time, the population changes because beneficial traits accumulate and
deleterious ones are weeded out.*

The Hall of Fame retains the best genomes across generations; new generations
are seeded primarily from these elites (`seed_pct_elite`), with a smaller
contribution from the broader population (`seed_pct_pack`) and random jitter
(`seed_pct_jitter`). The population is not the same at generation 100 as it
was at generation 1 — not because anything "learned", but because the
distribution of heritable variants has shifted.

**You can see this** by comparing the mean fitness of generation 1 and
generation 50 in the fitness chart. It will be visibly higher in most runs.

### The conclusion: adaptation without a designer

None of these mechanisms knows what a "good" genome looks like. Nobody tells
the creatures to avoid walls, forage efficiently, or flee predators. Those
behaviours emerge because the creatures that happen to exhibit them leave
more offspring, and the creatures that don't, don't. Over many generations,
the population is *adapted* to its environment — not because anything
intended it, but because selection has no other possible outcome when the
four postulates above hold.

This is the central insight of Darwin's work, and this project is an
interactive demonstration of it. Running the simulation for a few thousand
generations is the digital equivalent of watching a Galápagos finch's beak
change shape in response to a new seed type — but at a timescale of minutes
rather than millions of years.

### Secondary evolutionary phenomena that emerge

Beyond the four postulates, this project produces several well-known
evolutionary dynamics as emergent side effects:

- **Red Queen dynamics.** As rogues evolve better hunting, non-rogues evolve
  better evasion, and vice versa. Neither side stays ahead for long.
- **r/K selection.** A creature that reproduces at the bare
  `reproduce_threshold` of energy is r-selected; one that hoards energy and
  reproduces rarely is K-selected. Both strategies appear in the population
  and their relative frequencies fluctuate with resource availability.
- **Competitive exclusion.** Two kin lineages competing for the same food
  type will eventually see one go extinct locally — unless they partition
  the resource by evolving different foraging biases.
- **Character displacement.** Lineages that coexist over many generations
  tend to diverge in hue, because hue determines who they predate and who
  they cooperate with. Diverging in hue is a way to avoid predation and
  gain kin.
- **Genetic drift.** In small populations, non-adaptive traits can fix by
  chance. The `seed_pct_jitter` parameter explicitly injects this kind of
  randomness into each generation's starting population.
- **Mass extinction and adaptive radiation.** Press the asteroid button and
  you'll kill ~95% of the population at once. What survives is the tail of
  the fitness distribution — and the generations that follow often explore
  morphospace much more broadly than they did before, because the intense
  competition of the pre-impact equilibrium is gone.

These are not scripted. They are emergent consequences of the rules. That is
the entire point.

---

## Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│  Visualiser window  │         │  Dashboard window   │
│  (pywebview)        │         │  (pywebview)        │
└──────────┬──────────┘         └──────────┬──────────┘
           │  WebSocket                    │  WebSocket
           └────────────┬──────────────────┘
                        ▼
              ┌─────────────────────┐
              │  aiohttp server     │  Core/server.py
              │  + CommandRouter    │  Core/command_router.py
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  ColonyManager      │  Brains/manager.py
              │  (one colony per    │
              │   goal)             │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  Trainer + World    │  Brains/trainer.py
              │  (60 Hz loop)       │  Simulation/world.py
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  Population (torch) │  Brains/population.py
              │  + Registry (I/O)   │  Brains/registry.py
              └─────────────────────┘
```

`Main.py` starts the backend on a background thread, launches the two
windows on the main thread (pywebview requires this), and waits for both
windows to close before shutting down the server.

Each **colony** is a (world, population, trainer, registry) tuple dedicated
to a single goal. Only one colony is active at a time; switching goals
switches colonies. This lets you keep the elite genomes for all four goals
alive in parallel without them competing for selection pressure.

The `legacy/` directory contains an old C++ implementation of the simulation.
It is not built or run. It is preserved for reference only.

---

## Directory layout

```text
This Project/
├── Run.bat                     launcher
├── Install Deps.bat            one-time setup
├── Main.py                     entry point
├── config.json                 runtime tuning
├── requirements.txt            pip dependencies
│
├── Core/                       backend infrastructure
│   ├── config_loader.py        hot-reloadable config
│   ├── command_router.py       WebSocket command dispatch
│   ├── server.py               aiohttp HTTP + WS
│   ├── recorder.py             CSV export buffer
│   ├── radiation.py            WiFi scanner
│   ├── logger.py               event log
│   ├── shutdown.py             graceful shutdown
│   ├── window_manager.py       pywebview launcher
│   └── paths.py                relative path resolution
│
├── Simulation/                 pure-Python physics
│   ├── world.py                arena, tick loop
│   ├── creature.py             creature dataclass
│   ├── item.py                 food / poison
│   ├── physics.py              geometry helpers
│   ├── rules.py                collision interactions
│   ├── actions.py              god-mode drops
│   ├── events.py               tick event types
│   └── serialize.py            world → JSON
│
├── Brains/                     neuroevolution
│   ├── manager.py              colony manager
│   ├── population.py           population of nets
│   ├── network.py              net construction + mutation
│   ├── evolution.py            selection and crossover
│   ├── fitness.py              fitness scoring
│   ├── goals.py                goal definitions
│   ├── registry.py             Nural/ artifact I/O
│   └── trainer.py              the 60 Hz loop
│
├── Web/                        frontend
│   ├── shared/                 fonts, themes, helpers
│   ├── Visualiser/             arena render
│   └── Dashboard/              stats, controls, console
│
├── Nural/                      persisted brains
│   ├── Most_Food/
│   ├── Longest_Lifespan/
│   ├── Most_Reproductions/
│   └── Kills/
│
├── Assets/                     fonts and icons
├── runtime/venv/               local virtualenv
├── logs/                       event log + CSV
├── saves/                      save slots
└── legacy/                     archived C++ implementation
```

`runtime/venv/`, `logs/`, `saves/`, and the contents of `Nural/` are all
generated at runtime. Only the source directories should be versioned or
copied between machines.

---

## Configuration reference

`config.json` is read at startup and on `reload_config` command. Live-safe
values can also be modified from the Dashboard sliders. Full listing with
defaults:

### Arena

| Key | Default | Effect |
|---|---|---|
| `arena_size` | 800 | Arena edge length in world units |
| `max_creatures` | 50 | Hard population cap |
| `num_items` | 100 | Target item density at seed |
| `initial_population` | 15 | Creatures seeded at world start |
| `initial_poison_ratio` | 0.25 | Fraction of seeded items that are poison |
| `autosave_every_ticks` | 2000 | Autosave cadence |

### Creature biology

| Key | Default | Effect |
|---|---|---|
| `max_vision` | 150.0 | Daytime vision range |
| `food_energy` | 60.0 | Energy gained per food item |
| `poison_damage` | 40.0 | Energy lost per poison item at full potency |
| `reproduce_threshold` | 160.0 | Energy needed to split |
| `child_energy` | 80.0 | Energy granted to offspring |
| `base_metabolism` | 0.1 | Energy lost per tick at rest |
| `speed_metabolism` | 0.1 | Extra energy per tick per unit speed |
| `baseline_health` | 300.0 | Starting energy |

### Item dynamics

| Key | Default | Effect |
|---|---|---|
| `food_decay_ticks` | 400 | Food lifetime |
| `poison_decay_ticks` | 600 | Poison lifetime |
| `poison_potency_decay` | 0.0025 | Potency loss per tick |
| `food_spawn_per_tick` | 0.05 | Probability of one new item per tick |
| `poison_spawn_share` | 0.20 | Probability a new spawn is poison |
| `immigration_per_tick` | 0.005 | Probability one immigrant appears per tick |

### Interactions

| Key | Default | Effect |
|---|---|---|
| `hgt_chance` | 5 | % chance of HGT per contact |
| `kin_threshold_hue` | 20.0 | Max hue difference for kin |
| `predation_steal` | 15.0 | Energy stolen on predation |

### Evolution

| Key | Default | Effect |
|---|---|---|
| `mutation_base` | 5 | % of weights perturbed per generation |
| `mutation_severity` | 0.1 | Magnitude of each point mutation |
| `mutation_sigma` | 0.05 | Std-dev of Gaussian mutation |
| `saltation_rate` | 0.05 | % chance of large-effect mutation |
| `seed_pct_elite` | 0.50 | Fraction of next gen from elites |
| `seed_pct_pack` | 0.30 | Fraction from general population |
| `seed_pct_jitter` | 0.04 | Random jitter fraction |
| `crossover_rate` | 0.11 | Same-hue crossover probability |
| `crossover_rate_cross_hue` | 0.01 | Cross-hue crossover probability |
| `crossover_min_energy` | 120.0 | Energy floor for crossover |
| `crossover_energy_cost` | 20.0 | Energy cost of crossover |
| `cross_hue_mutation_mult` | 2.0 | Mutation multiplier for hybrids |

### Idle penalty

| Key | Default | Effect |
|---|---|---|
| `idle_speed_threshold` | 0.35 | Speed below which creature is considered idle |
| `idle_fitness_penalty` | 0.30 | Fitness drain per idle tick |

### Rogues

| Key | Default | Effect |
|---|---|---|
| `rogue_spawn_chance` | 0.5 | % of births that produce rogues |
| `rogue_damage_per_tick` | 4.0 | Contact damage |
| `rogue_counter_damage` | 2.0 | Damage taken back |
| `rogue_kill_heal` | 60.0 | Energy from a successful kill |
| `rogue_poison_heal` | 25.0 | Energy from poison |
| `rogue_food_damage` | 15.0 | Energy lost from food |
| `rogue_gangup_vision` | 180.0 | Radius for gang-up behaviour |
| `rogue_gangup_weight` | 0.7 | Gang-up weighting |

### Radiation

| Key | Default | Effect |
|---|---|---|
| `radiation_mult_speed` | 0.02 | Metabolism multiplier per rad unit |
| `radiation_drift_threshold` | 5 | Rad level at which positional jitter starts |
| `radiation_rain_threshold` | 10 | Rad level at which poison begins to fall |

### WiFi scanner

| Key | Default | Effect |
|---|---|---|
| `wifi_scan_enabled` | true | Enable periodic WiFi scan |
| `wifi_scan_interval_ms` | 4000 | Scan interval |
| `wifi_blacklist` | [] | SSIDs to ignore |
| `wifi_sensitivity` | 1.0 | Score scaling |
| `wifi_rssi_floor` | -85 | dBm below which signals are ignored |
| `wifi_rssi_ceiling` | -35 | dBm above which signals are capped |

### Neural architecture

| Key | Default | Effect |
|---|---|---|
| `network_hidden_depth` | 2 | Hidden layers |
| `network_hidden_width` | 64 | Units per hidden layer |
| `network_input_size` | 42 | Sensory inputs |
| `network_output_size` | 2 | Motor outputs |
| `device` | "cpu" | Torch device |

### Goals and fitness

| Key | Default | Effect |
|---|---|---|
| `active_goal` | "Most_Food" | Which colony drives the sim |
| `goal_list` | [four names] | Available goals |
| `fitness_w_food` | 0.5 | Weight of food eaten |
| `fitness_w_lifespan` | 1.0 | Weight of ticks lived |
| `fitness_w_reproductions` | 2.0 | Weight of offspring |
| `fitness_w_kills` | 3.0 | Weight of kills |

### Performance

| Key | Default | Effect |
|---|---|---|
| `target_tick_hz` | 60 | Simulation ticks per second |
| `day_cycle_ms` | 60000 | Wall-clock ms per full day |

---

## Controls and commands

### Keyboard

| Key | Action |
|---|---|
| `Space` | Pause / resume |
| `A` | Trigger asteroid strike |
| `R` | Reset world |
| `S` | Save to slot |
| `↑` / `↓` | Nudge speed by 0.25× |
| `Shift + ↑` / `↓` | Nudge speed by 1× |
| `1` / `2` / `3` / `4` | Quick speed presets |

### Mouse (Visualiser)

| Input | Action |
|---|---|
| Left click | Drop resources at cursor |
| Right click | Attract all creatures to cursor |
| Right click while inspecting | Pathfind inspected creature to cursor |
| Click a creature | Open inspector panel |

### Console

The Dashboard console accepts text commands. Type `help` for the list.
Highlights:

```
pause               freeze the world
resume              unfreeze
speed 2             run at 2× wall-clock speed
feed 10             drop 10 food items across the arena
poison 5            drop 5 poison items
asteroid            kill ~95% of the population
spawn 180 5         spawn 5 creatures with hue ≈ 180
attract 400 400     attract all creatures to arena centre
group 90            attract all creatures in hue bin around 90
tp 12 400 400       teleport creature 12 to (400, 400)
kill 12             kill creature 12
kill_all            kill everything (triggers rollover)
set_attr 12 energy 500
set max_vision 200  tune a live parameter
switch_goal Kills   change the active selection goal
load_brain gen:0007 all     load a saved genome into all
list_brains         list saved genomes
remove all          clear all items from the arena
```

---

## Python concepts used in this codebase

If you are not a deep Python user, this section is a guided tour of every
non-obvious Python feature that appears in the source. Skim it once and the
rest of the code will read much more easily.

### Imports across folders (packages)

A Python folder containing an `__init__.py` file is called a **package**. The
subfolders of this project (`Core/`, `Simulation/`, `Brains/`) are all
packages. When you write:

```python
from Core import paths
```

you are saying: "look for a module named `paths.py` inside the package
`Core/`, and bring it into this file under the name `paths`". This is how
`Main.py` reaches `Core.paths.ROOT` without hard-coding a file path — Python
resolves it relative to the project root because `Main.py` sits at the root
and `Core/` is a subfolder.

The three common import forms are:

```python
import math                      # import the whole module
from math import sqrt            # import one name from a module
from Simulation.world import World   # import a class from a subpackage
```

Relative imports use a leading dot to mean "from the current package":

```python
from .creature import Creature   # from the same folder as this file
from ..core import paths         # from a sibling package (one level up)
```

You will see both forms in the codebase. `Brains/trainer.py` uses
`from .world import World` because `world.py` is in the sibling `Simulation/`
package; it is reached via the package root that `Main.py` establishes.

### Type hints — `def func() -> None:`

Python is dynamically typed: you can write `x = 5` and later `x = "hello"`
without error. But you can *annotate* what a variable or function is
supposed to hold, and these annotations are what make a codebase readable
even after months away from it.

A function with a type hint looks like this:

```python
def greet(name: str) -> str:
    return "Hello, " + name
```

The `name: str` says "the caller should pass a string." The `-> str` says
"this function returns a string."

`-> None` means **the function returns nothing**. It performs an action (a
side effect) but hands no value back to the caller:

```python
def log_event(kind: str, msg: str) -> None:
    print(f"[{kind}] {msg}")
```

You will see `-> None` constantly in `Core/`, `Simulation/`, and `Brains/`
— anywhere a function is a pure side effect (writing to a file, updating
state, broadcasting a WebSocket message).

Type hints can describe much richer shapes than just `str` and `int`:

```python
def hue_diff(a: float, b: float) -> float: ...
def worlds(goal_list: list[str] | None = None) -> None: ...
def step(self, actions: np.ndarray) -> List[Event]: ...
```

- `list[str]` means "a list of strings" (Python 3.9+ syntax).
- `str | None` means "a string, or nothing at all" (Python 3.10+ syntax).
  The older equivalent is `Optional[str]`, and you will find both in the
  codebase depending on when the file was last touched.
- `List[Event]` (capital L, imported from `typing`) is the pre-3.9 way of
  spelling `list[Event]`. Both are still valid.

### `from __future__ import annotations`

You will see this at the top of every substantial file:

```python
from __future__ import annotations
```

This tells Python to treat all type annotations as *strings* until they are
actually needed. Two practical benefits:

1. You can write `list[Event]` and `Event | None` on Python 3.7+ without
   import errors, even though those syntaxes were only officially added
   later.
2. Forward references — annotations that mention a class defined further
   down the file — just work, without needing quote marks.

It is a small piece of boilerplate that makes the rest of the file cleaner.
Do not remove it.

### `@dataclass` — lightweight record types

A `@dataclass` decorator automatically writes `__init__`, `__repr__`, and
comparison methods for a class whose only job is to hold a bundle of fields:

```python
from dataclasses import dataclass

@dataclass
class Creature:
    x: float = 0.0
    y: float = 0.0
    angle: float = 0.0
    energy: float = 0.0
    alive: bool = False
```

This is equivalent to writing a full constructor by hand. It is used for
`Creature`, `Item`, and every event type in `Simulation/events.py`. It is a
way of saying "this thing is a struct, treat it as a bag of named values."

### `if TYPE_CHECKING:` — avoiding circular imports

Occasionally a module needs to refer to a class defined elsewhere, but
importing it at the top of the file would cause a circular dependency (A
imports B, B imports A). The pattern used in `Simulation/rules.py` is:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from .world import World

def handle_collisions(world: "World") -> List[object]:
    ...
```

`TYPE_CHECKING` is `False` at runtime, so the `import` inside the `if` never
actually happens — it exists only for type checkers. The string `"World"` in
the function signature is a forward reference that a type checker will
resolve. At runtime, Python just sees the string and moves on.

### `async` / `await` — asynchronous code

The backend uses `asyncio` throughout. The two keywords you will see are:

```python
async def _main(self) -> None:
    port = await self.server.start()
    await self.manager.start()
```

- `async def` declares a **coroutine**, a function that can pause and resume.
- `await` marks a point where the coroutine hands control back to the event
  loop while it waits for something (a network read, a timer, a WebSocket
  message).

Why bother? The server needs to handle multiple WebSocket clients, the tick
loop, and HTTP endpoints all at once. Without async, you would need threads.
With async, one thread handles everything cooperatively. This is why
`Main.py` has one backend thread running an entire `asyncio` loop — the
"concurrency" lives inside that loop, not in the OS thread scheduler.

A rule of thumb: if you need to call an `async def` function, you must
`await` it, and the caller must itself be `async def`.

### f-strings

String formatting like `f"[{kind}] {msg}"` is an **f-string**. The `f`
prefix tells Python to substitute any `{...}` expression with its runtime
value. It is the modern, preferred replacement for `.format()` and `%`
formatting. You will see them everywhere in `Core/`.

### Comprehensions — building a list or dict in one line

```python
alive_creatures = [c for c in self.creatures if c.alive]
scores = {c.id: c.fitness for c in self.creatures if c.alive}
```

A **list comprehension** builds a list by iterating and filtering. A **dict
comprehension** does the same for dictionaries. Both are one-liners that
would otherwise be three or four lines of loop. If you see square brackets
containing `for` and `if`, that is a comprehension.

### `with` — context managers

```python
with open(path, "r", encoding="utf-8") as fh:
    data = json.load(fh)
```

The `with` statement guarantees that the file is closed when the block
exits — even if an exception is thrown. It is used for file I/O, for
PyTorch's `torch.no_grad()` scope, and for anything else that needs a
guaranteed setup/teardown pair.

### Optional values and `Optional[...]`

When a value may be missing, its type is annotated as `Optional[T]` or
`T | None`:

```python
self.port: int | None = None
def _free_item_slot(self) -> int:
    for i, it in enumerate(self.items):
        if not it.active:
            return i
    return -1
```

The `| None` suffix means "either an int or None." Type checkers will force
you to check for `None` before using the value, which catches a class of
bugs that would otherwise fail at runtime.

### Tuples and unpacking

```python
x, y, angle = bounce_bounds(x, y, angle, arena_size)
```

If a function returns a tuple (a fixed-length group of values), you can
**unpack** it into named variables in one line. This is used heavily in
`Simulation/physics.py` and `Simulation/world.py`.

### `pass` — the do-nothing statement

You will see `pass` occasionally where Python's syntax requires a statement
but no action is wanted:

```python
try:
    self._fh.close()
except Exception:
    pass
```

This is idiomatic "ignore this error silently" — used only when the failure
is genuinely unimportant.

### numpy arrays

`Simulation/world.py` uses numpy for anything vector-shaped:

```python
inputs = np.zeros((MAX_CREATURES, NUM_SENSORY), dtype=np.float32)
row[0:8] = food_rays
```

`np.zeros((50, 42))` is a 2D array of zeros with 50 rows and 42 columns.
Slicing (`row[0:8]`) writes to a contiguous span of columns in one line.
numpy is used rather than Python lists wherever the data is numeric and
regular, because it is 10 to 100 times faster for bulk operations. This is
one of the main reasons the simulation can run at 60 Hz on a laptop.

### PyTorch tensors

`Brains/` uses PyTorch for the neural networks. A `torch.Tensor` is like a
numpy array that can also live on a GPU and compute gradients — although in
this project, both are disabled (`device: "cpu"`, and evolution is done by
mutation rather than gradients, so no autograd is needed).

The main operations you will see:

```python
net = torch.nn.Sequential(...)      # build a feed-forward network
with torch.no_grad():               # disable gradient bookkeeping
    actions = net(inputs)           # forward pass, fast
```

`torch.no_grad()` is important for performance: without it, PyTorch would
build a computation graph for backprop that is never used. With it, forward
passes are several times faster.

---

## Development

### Python

```bat
python -m py_compile Main.py Core\*.py Brains\*.py Simulation\*.py
```

### JavaScript

```bat
node --check Web\Dashboard\Analytics.js
node --check Web\Dashboard\Controls.js
node --check Web\Dashboard\Muller.js
```

### Simulation without the UI

You can drive `Simulation.world.World` from a Python shell:

```python
from Core.config_loader import get_config
from Simulation.world import World
import numpy as np

world = World(get_config().view())
actions = np.zeros((50, 2), dtype=np.float32)  # all-zero: no movement
for _ in range(1000):
    world.step(actions)
print(world.alive_count, world.tick)
```

This is useful for testing physics changes without launching the full app.

### Adding a new sensory input

1. Increase `network_input_size` in `config.json`.
2. Extend `World.collect_sensory_inputs()` to fill the new slot.
3. Delete any existing checkpoints in `Nural/` — the old weights are no
   longer loadable against the new input dimension.

---

## AI assistance

AI was used as part of the development process in three distinct roles.

**Writing and styling the frontend.** The CSS for the Visualiser and
Dashboard was written and iterated with AI assistance. Themes, responsive
layout, the stacked Muller plot, the neural visualisers, and the console
styling all went through AI editing passes. This is the layer where AI
contributes most directly to the final product.

**Performance optimisation.** AI was used to identify and fix the hot paths
in the simulation loop. Concretely, it helped with:

- switching the sensory input builder from per-creature Python loops to
  numpy array slices, which is why the world can tick at 60 Hz with 50
  creatures and 240 items on a laptop CPU;
- hoisting repeated attribute lookups (`self.cfg.x`) out of tight loops
  into local variables, a small change that measurably reduces per-tick
  overhead;
- wrapping the neural forward pass in `torch.no_grad()` and batching the
  population into one tensor rather than 50 individual nets, which cut the
  inference cost by roughly an order of magnitude;
- identifying that per-generation checkpoint writes were serialising the
  tick loop, and moving them off the critical path;
- replacing per-creature `math.sqrt` calls with squared-distance comparisons
  in collision detection, which is why the collision sweep is O(n²) but
  still fast enough at n=50.

These are the kinds of changes that do not alter the simulation's semantics
but do determine whether it runs at 60 Hz or 6 Hz. AI was used as a
reviewer and pair-programmer for them.

**Comments, syntax checking, and documentation.** AI added docstrings and
inline comments across the source, checked syntax during development, and
produced the initial draft of this README (which was then reviewed and
edited). The "Python concepts used in this codebase" section above is
written with AI assistance specifically for readers who are new to Python
and want a walking tour of the language features that appear in the source.

The project's architecture, simulation rules, and neural network design were
reviewed and tested after every AI-assisted change. The source files remain
the authoritative place to verify how the simulation actually behaves.

The simulation itself, of course, is not written by an AI — it *is* an AI,
or more precisely, a population of tiny AIs whose behaviour was shaped not by
a human designer but by millions of years of simulated selection pressure,
compressed into minutes of wall-clock time. That is the point.

> Note From me, the dev - `PB ( 9A )` :  
>   
> most of this readme and the `Css` was `"made"` using Generative Artifitial Inteligence.  
> The syntax checking and optimisations to performance was also added via generative ai's `"help"`  
> There is no specific model used to do each task, and there is a very small chance that some information coded was wrong, ie.  
> Slightly buggy `Css` for cards in the dashboard needed manual tweaking with the `Css`.
> I had a lot of fun making, and modifying this project. the longer this simulation runs, the faster and better our specimins get...   
