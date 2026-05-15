# Configuration Reference

This page provides a consolidated list of the most commonly used configuration parameters in BookSim.

## General Simulation

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sim_type` | string | `latency` | Type of simulation (`latency`, `throughput`). |
| `sim_count` | int | 1 | Number of simulations to perform. |
| `seed` | int | 0 | Random seed. |
| `sample_period`| int | 1000 | Measurement period duration in cycles. |
| `warmup_periods`| int | 3 | Number of periods to discard for warmup. |

## Network and Topology

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `topology` | string | `torus` | Topology name. |
| `k` | int | 8 | Radix (nodes per dimension). |
| `n` | int | 2 | Number of dimensions. |
| `c` | int | 1 | Concentration (nodes per router). |
| `routing_function` | string | `none` | Routing algorithm name. |

## Router Architecture

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `router` | string | `iq` | Router model (e.g., `iq`). |
| `num_vcs` | int | 16 | Number of virtual channels per port. |
| `vc_buf_size` | int | 8 | Flits per VC buffer. |
| `vc_allocator` | string | `islip` | VC allocator type. |
| `sw_allocator` | string | `islip` | Switch allocator type. |
| `internal_speedup` | float | 1.0 | Internal crossbar speedup factor. |

## Traffic and Injection

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `traffic` | string | `uniform` | Traffic pattern name. |
| `injection_rate` | float | 0.1 | Injection rate (packets/node/cycle). |
| `injection_process` | string | `bernoulli` | Injection model (`bernoulli`, `on_off`). |
| `packet_size` | int | 1 | Number of flits per packet. |

## Logging and Debugging

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `print_activity` | int | 0 | Print fine-grained router activity. |
| `stats_out` | string | "" | Output file for statistics. |
| `watch_out` | string | "" | Output file for watched flits. |

---
*For a complete list of all parameters and their default values, refer to the [BookSimConfig](../../booksim/src/booksim_config.cpp#L38) constructor in the source code.*
