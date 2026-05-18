[<- Topology Index](README.md)

# Mesh / Torus In-Depth Topology Guide

The standard regular $N$-dimensional grid topologies (Mesh and Torus) are implemented in the [KNCube](../../booksim/src/networks/kncube.cpp#L45) class inside [kncube.cpp](../../booksim/src/networks/kncube.cpp).

---

## 1. Grid Sizing & Parameters

The constructor computes the overall grid size from the configuration:
-   `k` (`_k`): Radix (number of routers along each dimension).
-   `n` (`_n`): Dimension count.
-   `c` (`_c`): Concentration (default is 1).

### Sizing Formulas (`_ComputeSize`)
```cpp
_size     = powi( _k, _n );    // Total routers
_channels = 2 * _n * _size;    // Total inter-router channels
_nodes    = _size;             // Total processor nodes
```

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/kncube.cpp#L70) method constructs the router grid, configures its channel latencies, and attaches terminal processor nodes.

```mermaid
graph LR
    LeftNode["Left Node (L)"] -- "_LeftChannel" --> Current["Current Node (N)"]
    RightNode["Right Node (R)"] -- "_RightChannel" --> Current
    Current -- "_RightChannel" --> RightNode
    Current -- "_LeftChannel" --> LeftNode
```

### Step 2.1: Router Indexing and Naming
Each router node is allocated with a unique coordinate-based string identifier corresponding to its $N$-dimensional grid position:
```cpp
if ( _k > 1 ) {
  for ( int dim_offset = _size / _k; dim_offset >= 1; dim_offset /= _k ) {
    router_name << "_" << ( node / dim_offset ) % _k;
  }
}
```
*Example:* In a 3D grid ($k=4, n=3$), node index `37` maps to coordinates `( node / 16 ) % 4 = 2`, `( node / 4 ) % 4 = 1`, and `( node / 1 ) % 4 = 1`, resulting in the name `router_2_1_1`.

### Step 2.2: Router Allocation
Each switch is allocated as a standard input-queued router with a degree of $2n + 1$ (representing the left and right channels for each of the $n$ dimensions, plus $1$ terminal node interface):
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, 2*_n + 1, 2*_n + 1 );
```

### Step 2.3: Inter-Router Connections & Latency
For each dimension `dim` from `0` to `_n - 1`, the builder establishes connections to its neighbors:
1.  **Neighbor Identification**: Neighbors are identified via modulo-radix arithmetic:
    *   `left_node = _LeftNode(node, dim)`
    *   `right_node = _RightNode(node, dim)`
    *   *Wrap-around*: If a node is at coordinate $0$, its left neighbor wraps around to $\text{radix} - 1$. If at $\text{radix} - 1$, its right neighbor wraps around to $0$.
2.  **Torus vs. Mesh Wire Delay**:
    *   **Torus**: Inter-switch latency defaults to **2 cycles** due to wrap-around cable lengths.
    *   **Mesh**: Inter-switch latency defaults to **1 cycle**.
    *   This is evaluated dynamically: `int latency = _mesh ? 1 : 2;`.
3.  **Channel Indexing**:
    *   `_RightChannel(node, dim)` returns the channel index: `2 * _n * node + 2 * dim`.
    *   `_LeftChannel(node, dim)` returns the channel index: `2 * _n * node + 2 * dim + 1`.
4.  **Channel Binding**:
    ```cpp
    // Add incoming channels from neighbors
    _routers[node]->AddInputChannel( _chan[right_input], _chan_cred[right_input] );
    _routers[node]->AddInputChannel( _chan[left_input], _chan_cred[left_input] );
    
    // Add outgoing channels to neighbors
    _routers[node]->AddOutputChannel( _chan[right_output], _chan_cred[right_output] );
    _routers[node]->AddOutputChannel( _chan[left_output], _chan_cred[left_output] );
    ```

### Step 2.4: Injection and Ejection
Lastly, processor nodes are connected to port $2n$:
```cpp
_routers[node]->AddInputChannel( _inject[node], _inject_cred[node] );
_routers[node]->AddOutputChannel( _eject[node], _eject_cred[node] );
_inject[node]->SetLatency( 1 );
_eject[node]->SetLatency( 1 );
```

---

## 3. Resilience and Fault Injection

The `kncube` topology fully implements [InsertRandomFaults](../../booksim/src/networks/kncube.cpp#L231) to evaluate routing resilience:
-   It accepts `link_failures = <int>` and seeds the failure generator using the `fail_seed` value.
-   It identifies internal channels (excluding edge channels in non-wrap-around mesh configs) and introduces faults using the `OutChannelFault(node, chan)` method, marking the designated link as inactive.
