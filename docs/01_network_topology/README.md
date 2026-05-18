[Index](../index.md)

# Network Topology

BookSim supports a variety of network topologies, ranging from standard regular structures to custom arbitrary graphs. The topology is selected using the `topology` parameter in the configuration file.

## Available Topologies

- **Mesh / Torus ([kncube.cpp](../../booksim/src/networks/kncube.cpp#L45))**:
  - `k`: The radix (number of nodes per dimension).
  - `n`: The number of dimensions.
  - `c`: Concentration (number of nodes per router).
  - Example: `topology = mesh; k = 8; n = 2;` defines an 8x8 mesh.

- **Concentrated Mesh ([cmesh.cpp](../../booksim/src/networks/cmesh.cpp#L56))**:
  - A mesh with multiple nodes per router, often used for on-chip networks.
  - Supports additional layout parameters like `x`, `y` for router placement.

- **Fat-Tree ([fattree.cpp](../../booksim/src/networks/fattree.cpp#L58))**:
  - A hierarchical tree structure.
  - Parameters: `k` (radix), `n` (height).

- **Dragonfly ([dragonfly.cpp](../../booksim/src/networks/dragonfly.cpp#L149))**:
  - A high-radix topology with groups of routers.

- **Butterfly ([fly.cpp](../../booksim/src/networks/fly.cpp#L37))**:
  - A multi-stage interconnection network.

- **Arbitrary Networks ([anynet.cpp](../../booksim/src/networks/anynet.cpp#L61))**:
  - Allows defining custom topologies via a text file.
  - Parameter: `network_file`.

## Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `topology` | The name of the topology to use. | `torus` |
| `k` | Number of nodes per dimension (radix). | 8 |
| `n` | Number of dimensions. | 2 |
| `c` | Concentration (nodes per router). | 1 |
| `use_noc_latency` | Adjusts latency based on physical distance. | 1 |

## Custom Topologies with `anynet`

The `anynet` topology reads the network structure from a file specified by `network_file`. Each line in the file defines a connection:
```
router <id> port <p> -> router <id> port <p>
```
Or for nodes:
```
node <id> -> router <id> port <p>
```
