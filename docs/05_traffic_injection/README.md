# Traffic Injection

Traffic injection defines how and when packets are introduced into the network by the source nodes.

## Injection Processes

The `injection_process` parameter defines the temporal distribution of packet injection.

- **Bernoulli (`bernoulli`)**: Packets are injected with a fixed probability `injection_rate` at each cycle.
- **On-Off (`on_off`)**: A bursty traffic model that toggles between "on" (active injection) and "off" (idle) states.
  - `burst_alpha`: Probability of switching from off to on.
  - `burst_beta`: Probability of switching from on to off.

## Traffic Patterns

The `traffic` parameter defines the spatial distribution (destination nodes).

| Pattern | Description |
|---------|-------------|
| `uniform` | Destinations are chosen uniformly at random. |
| `bitcomp` | Destination is the bitwise complement of the source ID. |
| `transpose` | Source $(i, j)$ sends to destination $(j, i)$. |
| `shuffle` | Destination is a cyclic left shift of the source ID bits. |
| `tornado` | Destinations are shifted by $(k/2 - 1)$ in each dimension. |
| `neighbor` | Destination is the immediate neighbor in the first dimension. |
| `hotspot` | Certain nodes receive a higher fraction of traffic. |

## Packet and Flit Configuration

- `packet_size`: Number of flits per packet.
- `injection_rate`: The rate at which traffic is injected (packets per node per cycle).
- `injection_rate_uses_flits`: If set to 1, the injection rate is interpreted as flits per cycle.

## Multiple Traffic Classes

BookSim supports multiple traffic classes (e.g., for different priority levels or message types).
- `classes`: Number of traffic classes.
- Many parameters (like `injection_rate` and `priority`) can be specified as a list (e.g., `injection_rate = {0.1, 0.05}`) to apply different values to each class.
