# BookSim Documentation Index

This documentation is organized by subsystem and simulator functionality.

---

# Sections

## 1. Network Topology

Documentation related to network structure and topology generation.

➡️ [Open Section](01_network_topology/README.md)

Topics:
- topology
- k
- n
- mesh
- torus
- fat-tree

---

## 2. Routing

Routing algorithms and routing-related parameters.

➡️ [Open Section](02_routing/README.md)

Topics:
- routing_function
- dimension-order routing
- adaptive routing
- valiant routing
- UGAL
- NOQ

---

## 3. Flow Control & Buffers

Virtual channels, buffering, and credit flow.

➡️ [Open Section](03_flow_control_buffers/README.md)

Topics:
- num_vcs
- vc_buf_size
- wait_for_tail_credit
- credit flow
- HOL blocking

---

## 4. Router Architecture

Router pipeline and allocator internals.

➡️ [Open Section](04_router_architecture/README.md)

Topics:
- VC allocation
- switch allocation
- speculative allocation
- arbitration
- router delays
- pipeline stages

---

## 5. Traffic & Injection

Traffic generation and injection models.

➡️ [Open Section](05_traffic_injection/README.md)

Topics:
- traffic patterns
- injection_rate
- packet_size
- hotspot traffic
- transpose traffic
- traffic classes

---

## 6. Simulation Control

Simulation execution and convergence behavior.

➡️ [Open Section](06_simulation_control/README.md)

Topics:
- warmup periods
- sample periods
- convergence thresholds
- sim_type

---

## 7. Debugging & Tracing

Tracing infrastructure and debug outputs.

➡️ [Open Section](07_debugging_tracing/README.md)

Topics:
- watch_out
- stats_out
- TRACK_* outputs
- flit traces
- stored_flits.txt
- congestion visualization

---

## 8. Power Analysis

Power and wire modeling.

➡️ [Open Section](08_power_analysis/README.md)

Topics:
- sim_power
- tech_file
- Orion integration
- wire models

---

## 9. Reference

Reference material and configuration walkthroughs.

➡️ [Open Section](09_reference/README.md)

Topics:
- all parameter index
- example configurations
- experimental configs
