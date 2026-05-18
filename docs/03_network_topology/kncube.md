[<- Topology Index](README.md)

# Mesh / Torus Topology

Standard regular $N$-dimensional grid topologies (Mesh and Torus) are implemented in the [KNCube](../../booksim/src/networks/kncube.cpp#L45) class inside [kncube.cpp](../../booksim/src/networks/kncube.cpp).

---

## 1. Parameters & Sizing Configuration

-   **`k` (`_k`)**: Radix (number of routers along each dimension).
-   **`n` (`_n`)**: Dimension count.
-   **`c` (`_c`)**: Concentration factor (default is 1).

### Network Size Metrics
-   **Total Routers (`_size`)**: `powi(_k, _n)`
-   **Total Inter-router Channels (`_channels`)**: `2 * _n * _size`
-   **Total Terminals (`_nodes`)**: `_size`

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/kncube.cpp#L70) method instantiates routers and establishes inter-switch and terminal channel connections.

### 2.1 Router Allocation & Naming
Routers are allocated as standard input-queued switches with a uniform degree of $2n + 1$ (left/right grid ports per dimension, plus 1 local injection/ejection interface):
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, 2*_n + 1, 2*_n + 1 );
```
Switch names are resolved as coordinate representations (`router_coord0_coord1_...`) based on grid division:
```cpp
(node / dim_offset) % _k
```

### 2.2 Inter-Router Channels & Latency Configuration
For each dimension `dim` from `0` to `_n - 1`:
-   **Neighbor Lookups**: Neighbors are identified using `_LeftNode(node, dim)` and `_RightNode(node, dim)`.
-   **Channel Mapping**: Output channels are registered via `_RightChannel(node, dim)` and `_LeftChannel(node, dim)`. Input channels are registered via `_LeftChannel(right_neighbor, dim)` and `_RightChannel(left_neighbor, dim)`.
-   **Wire Delay**: When `use_noc_latency = 1` is configured, torus wrap-around links are assigned **2 cycles** latency, while standard mesh links are assigned **1 cycle** latency:
    ```cpp
    int latency = _mesh ? 1 : 2;
    ```

### 2.3 Injection and Ejection Interfaces
Terminal client nodes connect to port $2n$ on each switch. These injection/ejection channels are assigned a default latency of **1 cycle**.

---

## 3. Link Failure Injection

The topology supports random link faults at startup via `InsertRandomFaults(config)` using the `link_failures` configuration integer. Designated channels are disabled dynamically via the `OutChannelFault` lookup.
