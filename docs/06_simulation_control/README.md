# Simulation Control

Simulation control parameters determine how long a simulation runs and how statistics are collected.

## Simulation Types (`sim_type`)

- **Latency (`latency`)**: Measures the average latency for a fixed injection rate. The simulation runs until the latency converges.
- **Throughput (`throughput`)**: Measures the maximum sustained throughput.

## Simulation Phases

A typical simulation consists of two main phases:

1. **Warmup Phase**:
   - The network is allowed to reach a steady state.
   - `warmup_periods`: Number of sample periods to discard before starting measurements.
   - `warmup_thres`: Maximum relative change between periods to consider the system "warmed up".

2. **Measurement Phase**:
   - Statistics are collected over several sample periods.
   - `sample_period`: Duration of each measurement period in cycles.
   - `max_samples`: Maximum number of sample periods.
   - `stopping_thres`: Convergence criteria for ending the simulation.

## Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `sim_count` | Number of independent simulations to run. | 1 |
| `seed` | Random seed for reproducibility. | 0 |
| `latency_thres` | Threshold after which the network is considered unstable (deadlocked/saturated). | 500.0 |
| `print_csv_results` | Outputs a summary in CSV format. | 0 |

## Batch Simulation

For certain workloads, BookSim supports batch simulation where a fixed number of packets are injected and the time to complete the batch is measured.
- `batch_size`: Number of packets per batch.
- `batch_count`: Number of batches to run.
