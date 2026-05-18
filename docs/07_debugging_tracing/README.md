[<- Previous Page](../06_simulation_control/README.md) | [Index](../index.md)

# Debugging and Tracing

BookSim provides several tools for monitoring the state of the network and debugging performance issues.

## Activity Monitoring

- `print_activity`: If set to 1, prints a detailed log of buffer reads/writes and crossbar activity at each cycle. This is useful for identifying bottlenecks.
- `viewer_trace`: Generates a trace file that can be used with external visualization tools.

## Flit and Packet Watching

You can track specific flits or packets as they move through the network.

- `watch_file`: A file containing IDs of flits/packets to watch.
- `watch_flits`, `watch_packets`: Direct specification of IDs to watch.
- `watch_out`: The file where watch information is logged.

## Statistics and Output Files

- `[stats_out](stats_out.md)`: File where final statistics (average latency, throughput, etc.) are saved. The statistics collection and reporting logic is defined in [TrafficManager::WriteStats](../../booksim/src/trafficmanager.cpp#L1801).
- **Detailed Flow Tracking** (if compiled with `TRACK_FLOWS`):
  - `[injected_flits_out](injected_ejected_flits.md)` / `[ejected_flits_out](injected_ejected_flits.md)`
  - `[received_flits_out](sent_received_flits.md)`
  - `[stored_flits_out](stored_flits.md)` (useful for buffer occupancy analysis)
  - `[sent_flits_out](sent_received_flits.md)`
  - `outstanding_credits_out`

## Deadlock Detection

- `deadlock_warn_timeout`: Number of cycles a flit can be stalled before a deadlock warning is issued.

## Example Usage

To watch a specific flit ID 1234 and output to `debug.txt`:
```
watch_flits = 1234
watch_out = debug.txt
```
To print activity for performance analysis:
```
print_activity = 1
```
