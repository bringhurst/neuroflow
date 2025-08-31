# Neuroflow

**WARNING: This repo is an early work-in-progress. The contents of this readme are just a wish list of what I want this project to be! Come back later and something useful might exist here.**

_Composable, event-driven spiking neural circuits based on dataflow-style pipelines._

## Overview

Neuroflow is a lightweight framework for composing and visualizing spiking neural circuits. The library is built for interactive use (Jupyter notebooks, REPL) and experimentation that requires precise control over spike timing and structure.

Network training is mostly through hand-crafted hardwired circuits (backprop-style training with surrogate gradients is also possible, but not the focus).

A typical user will start by using the Python API for composition and visualization. Composable Groups let the user encapsulate reusable subcircuits (e.g., adders, flip-flops). After the network is defined, a pluggable runner (for example the current local Zig-backed runner) is selected to execute the Pathway. Distributed runners are a planned future direction.

## Core Abstractions

### Spike
- A spike is an event carrying a source, destination, and a time interval [start, end].
- Intervals support [uncertainty and temporal windows](https://blog.typeobject.com/posts/2025-stdp-meets-truetime/).
- Spikes may carry optional payloads.

Example:
```python
spike = nf.Spike(source="A", target="B", interval=(0.0, 0.1), payload=None)
```

### Neuron
- LIFNeuron: Leaky Integrate-and-Fire, used for most gates.
- DLIFNeuron: Dendritic LIF for nonlinear computation (e.g., XOR-like functionality).
- CustomNeuron: Users can subclass a base Neuron to implement arbitrary behavior.

All neuron implementations expose:

- Named inputs and outputs
- Parameters (threshold, leak, weights, delays)
- A `visualize()` hook for inline plots in notebooks
- A deterministic `process` step compatible with the runner

### Synapse
- Synapse represents a connection from a pre to a post element with weight and delay.
- Plasticity can be attached to synapses (e.g. STDP or reward-modulated rules).
- Synapses are first-class and can be inspected and serialized.

Example:
```python
syn = nf.Synapse(pre=A, post=neuron, weight=1.0, delay=0.5)
```

### Groups (Composable Subcircuits)
- Group is a reusable container for neurons, synapses, inputs, outputs, and nested groups.
- Groups have explicit named inputs and outputs so they can be composed and reused.
- Groups are the right level to encapsulate a half-adder, flip-flop, WTA module, or pattern detector.

Example:
```python
class HalfAdder(nf.Group):
    def __init__(self, name="HalfAdder"):
        super().__init__(name=name)
        # define inputs, neurons, internal wiring, and outputs
```

Groups can be added to Pathways and nested inside other groups.

### Pathway
- Pathway is the runtime graph: spike inputs, groups, neurons, synapses, windows, and outputs. Similiar to a pipeline in [Apache Beam](https://beam.apache.org).
- You build the Pathway, supply input spike streams, and run it for a given simulation window.
- The current runner executes the Pathway deterministically in a single process using a Zig backend.

### Windowing / Broadcast
- Windowing collects spikes across a temporal window for coincidence or pattern detection.
- Broadcast provides a global channel (e.g. a reward/dopamine signal) that selected neurons can subscribe to.

## Quickstart

```bash
git clone https://github.com/bringhurst/neuroflow.git
```

## Tutorials

### Reusable Half-Adder Group

A HalfAdder is a canonical example of a reusable Group.

```python
import neuroflow as nf

class HalfAdder(nf.Group):
    def __init__(self, name="HalfAdder"):
        super().__init__(name=name)

        # public inputs
        self.A = nf.SpikeInput(name="A")
        self.B = nf.SpikeInput(name="B")

        # internal neurons for SUM and CARRY (SUM implemented with DLIF for XOR-like behavior)
        self.sum_neuron = nf.DLIFNeuron(name="SUM", dendrite_configs=[...])
        self.carry_neuron = nf.LIFNeuron(name="CARRY", threshold=1.5, weights=[1.0, 1.0])

        # wiring
        self.connect(self.A, self.sum_neuron)
        self.connect(self.B, self.sum_neuron)
        self.connect(self.A, self.carry_neuron)
        self.connect(self.B, self.carry_neuron)

        # register group IO
        self.set_inputs([self.A, self.B])
        self.set_outputs([self.sum_neuron, self.carry_neuron])
```

Use the group in a Pathway:
```python
ha = HalfAdder()
pathway = nf.Pathway()
pathway.add_group(ha)

results = pathway.run(
    inputs={
        ha.A: [(0.0, 1.0)],
        ha.B: [(0.0, 1.0)]
    },
    max_time=2.0
)

ha.sum_neuron.visualize(results)
ha.carry_neuron.visualize(results)
```

### Composing Groups into Larger Circuits

Groups can be nested or connected to build larger modules (e.g. FullAdder built from two HalfAdders). Each group acts like a reusable component with explicit IO, enabling library-style sharing and unit testing.

### Visualization

- `neuron.visualize(results)` produces membrane traces, threshold lines, and spike markers.
- `group.visualize(results)` can show an overview of member neurons and their spikes.
- All visualizations are notebook-friendly via matplotlib/HTML hooks.

## FAQ

Q: Can I simulate a brain?  
A: No. Only small to medium experiments (up to a few thousand neurons) are currently supported. The design anticipates future distributed runners, which would be necessary for brain-scale workloads.

Q: Can I define my own neuron/synapse?  
A: Yes. Subclass the base `Neuron` or `Synapse` and register behavior. Include `visualize()` hooks for notebook use.

## Contributing

- See [CONTRIBUTING](CONTRIBUTING.md) for more information.
