# BookSim Documentation & Research Repository

Documentation, architecture notes, tracing formats, and experimental analysis for BookSim.

---

## Repository Goals

This repository aims to:

- Document BookSim configuration parameters
- Explain internal router/network architecture
- Link documentation directly to source code

---

## Repository Structure

```text
.
├── booksim/        # BookSim source code
├── docs/           # Documentation
└── README.md
```

---

## Documentation

Main documentation index:

➡️ [Open Documentation Index](docs/index.md)

---

## Major Sections

| Section | Description |
|---|---|
| Network Topology | Mesh, torus, dimensions, radix |
| Routing | Routing algorithms and path selection |
| Flow Control & Buffers | Virtual channels, credits, buffering |
| Router Architecture | Allocators, speculation, pipelines |
| Traffic & Injection | Traffic patterns and injection models |
| Simulation Control | Warmup, convergence, statistics |
| Debugging & Tracing | Trace outputs and debugging tools |
| Power Analysis | Orion and power modeling |
| Reference | Parameter index and config walkthroughs |

---

## Codebase Integration

Documentation pages directly reference BookSim source files and implementation locations.

Example:

```text
booksim/src/routers/iq_router.cpp
```

Many pages also link to exact implementation lines for easier navigation.

---

## Experimental Focus

This repository also contains:

- latency-throughput analysis,
- routing experiments,
- VC scaling studies,
- congestion observations,
- NOQ behavior analysis,
- tracing format specifications.

---

## Future Goals

- Interactive visualization GUI
- Flit-level animation support
- Auto-generated parameter database
- AI-assisted BookSim exploration
- Searchable documentation website

---

## Related Resources

- BookSim simulator
- NoC architecture research
- Router microarchitecture
- Network-on-Chip simulation
