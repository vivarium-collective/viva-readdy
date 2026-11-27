# viva-readdy

[![tests](https://github.com/vivarium-collective/viva-readdy/actions/workflows/tests.yml/badge.svg)](https://github.com/vivarium-collective/viva-readdy/actions/workflows/tests.yml)

Engine-provider package (`pbg_readdy`): a [process-bigraph](https://github.com/vivarium-collective/process-bigraph)
wrapper for [ReaDDy](https://readdy.github.io), a particle-based reaction-diffusion simulator.

This repo ships **only the reusable Python package** — the `ReaDDyProcess` Process
and the `make_readdy_document()` composite factory — so other workspaces can depend
on it and compose ReaDDy with other biological models. Its demo studies and
investigation (`readdy-demo`, `readdy-demo-with-viz`) now live in the
**[viva-showcase](https://github.com/vivarium-collective/viva-showcase)** workspace,
which pins this package for its engine.

> **ReaDDy engine:** `import pbg_readdy` works without the ReaDDy engine installed —
> the engine is imported lazily and is only needed when a simulation actually runs.
> ReaDDy is typically conda-only (`conda install -c readdy readdy`) and not reliably
> pip-installable.

## Installation

```bash
python -m venv .venv
source .venv/bin/activate

pip install -e .          # package
pip install -e ".[dev]"   # + tests

# the engine itself (conda recommended), only needed to run simulations:
conda install -c readdy readdy
```

## Quick start

```python
import numpy as np
from process_bigraph import Composite, allocate_core
from process_bigraph.emitter import RAMEmitter
from pbg_readdy import ReaDDyProcess, make_readdy_document

core = allocate_core()
core.register_link('ReaDDyProcess', ReaDDyProcess)
core.register_link('ram-emitter', RAMEmitter)

positions = (np.random.random((30, 3)) * 8 - 4).tolist()
doc = make_readdy_document(
    box_size=[10., 10., 10.],
    species={'A': 1.0, 'B': 0.5},
    reactions=[{'descriptor': 'fusion: A +(2) A -> B', 'rate': 5.0}],
    potentials=[{
        'type': 'harmonic_repulsion',
        'species1': 'A', 'species2': 'A',
        'force_constant': 10., 'interaction_distance': 1.5,
    }],
    initial_particles={'A': positions},
    timestep=0.005,
    observe_stride=50,
    interval=1.0,
)

sim = Composite({'state': doc}, core=core)
sim.run(5.0)
print(sim.state['stores']['particle_counts'])
```

Composites are addressed as `pbg_readdy.composites.*`.

## API

### `ReaDDyProcess`

A time-driven `Process` bridging ReaDDy's Brownian-dynamics engine.

| Port | Type | Direction | Description |
|------|------|-----------|-------------|
| `particle_counts` | `map[integer]` | output | Count per species |
| `total_particles` | `integer` | output | Total particle count |
| `positions` | `list` | output | Particle positions `[[x,y,z], ...]` |
| `energy` | `float` | output | Total potential energy |
| `time` | `float` | output | Simulation time |

Key config: `box_size`, `periodic`, `kbt`, `timestep`, `species`
(name → diffusion constant), `reactions`, `potentials`, `initial_particles`,
`observe_stride`, `reaction_handler`.

Reaction formats:

```python
# Descriptor string (fusion, fission, decay, conversion)
{'descriptor': 'fusion: A +(2) A -> B', 'rate': 5.0}

# Enzymatic (catalyst preserved)
{'method': 'enzymatic', 'name': 'eat', 'catalyst': 'B',
 'type_from': 'A', 'type_to': 'B', 'rate': 1.0, 'educt_distance': 3.0}
```

Potential types: `harmonic_repulsion`, `lennard_jones`, `weak_interaction`,
`screened_electrostatics` (see `pbg_readdy/processes.py`).

Extra readout methods: `get_trajectory_data()` and
`get_position_snapshots(stride=None)`.

### `make_readdy_document()`

Factory that builds a ready-to-run composite document wiring a `ReaDDyProcess`,
its stores, and a RAM emitter.

## Architecture

ReaDDy manages its own internal particle state; the wrapper uses the **bridge
pattern** — lazily build the ReaDDy system (species, reactions, potentials), set
initial particles, run `simulation.run()` per `update(state, interval)`, and read
observables back through callbacks. The callback approach (rather than trajectory
files) lets `update()` be called repeatedly, so the process composes with others
in a `Composite`.

## Demo

```bash
source .venv/bin/activate
PYTHONPATH=. python demo/demo_report.py   # requires the ReaDDy engine
```

Generates `demo/report.html` — an interactive report with four simulations
(actin treadmilling, Lotka-Volterra, living polymers, crowded diffusion), each
with a 3D particle viewer, Plotly charts, and a bigraph diagram.

## License

MIT
