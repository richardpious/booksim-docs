[<- Topology Index](README.md)

# Dragonfly In-Depth Topology Guide

The Dragonfly hierarchical topology features tightly-connected groups of routers joined by long global channels. In BookSim, this is implemented in the [DragonFlyNew](../../booksim/src/networks/dragonfly.cpp#L149) class inside [dragonfly.cpp](../../booksim/src/networks/dragonfly.cpp).

> [!IMPORTANT]
> To use this topology, you must specify `topology = dragonflynew` in your configuration file. Using `topology = dragonfly` is not supported.

---

## 1. Parameters & Group Architecture

The Dragonfly topology scales using two main parameters:
-   `k` (`_p`): Switch local radix (defines the number of local terminal ports).
-   `n` (`_n`): Group dimension (must be set to `1` as higher-dimensional groups are not supported).

### Topology Sizing (`_ComputeSize`)
Based on $p$ (radix) and $n=1$, the group and network dimensions are calculated as follows:
-   **Routers per Group (`_a`)**: `2 * _p`
-   **Group Count (`_g`)**: `_a * _p + 1`
-   **Total Network Nodes (`_nodes`)**: `_a * _p * _g`
-   **Total Switches (`_num_of_switch`)**: `_nodes / _p`
-   **Total Inter-Router Channels (`_channels`)**: `_num_of_switch * (_k - _p)`
-   **Switch Total Radix (`_k`)**:
    ```cpp
    _k = _p + _p + 2 * _p - 1;
    ```
    This total radix $k$ is composed of:
    *   `_p` local terminal ports (injection/ejection)
    *   `_p` global inter-group ports (optical links)
    *   `2 * _p - 1` local intra-group ports (inter-router links inside the group)

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/dragonfly.cpp#L215) method allocates switches and wires three distinct classes of links:

```text
                  Dragonfly Hierarchical Architecture
       +-------------------------------------------------------+
       |                  [Group A]                            |
       |  +------------+  Intra-group  +------------+          |
       |  |  Switch 0  | <===========> |  Switch 1  |          |
       |  +------------+ (10 cycles)   +------------+          |
       |        ||                           ||                |
       +--------||---------------------------||----------------+
                || Global Inter-group link   || Global Inter-group link
                || (100 cycles latency)      || (100 cycles latency)
       +--------||---------------------------||----------------+
       |        ||                           ||                |
       |  +------------+  Intra-group  +------------+          |
       |  |  Switch X  | <===========> |  Switch Y  |          |
       |  +------------+ (10 cycles)   +------------+          |
       |                  [Group B]                            |
       +-------------------------------------------------------+
```

### Step 2.1: Switch Allocation
Switches are allocated with the computed radix `_k`:
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, _k, _k );
```

### Step 2.2: Local Processor Node Connections
Processor nodes connect to the first `_p` ports of each router. These injection and ejection channels are configured with a standard latency of **1 cycle**:
```cpp
for ( int cnt = 0; cnt < _p; ++cnt ) {
  c = _p * node + cnt;
  _routers[node]->AddInputChannel( _inject[c], _inject_cred[c] );
  _routers[node]->AddOutputChannel( _eject[c], _eject_cred[c] );
}
```

### Step 2.3: Connecting Local Intra-Group Links
Every router connects to all other `2 * _p - 1` routers within its local group. Under the `DRAGON_LATENCY` compilation macro, these links are configured with a hardcoded latency of **10 cycles**:
```cpp
for ( int dim = 0; dim < _n; ++dim ) {
  for ( int cnt = 0; cnt < (2*_p - 1); ++cnt ) {
    _output = (2*_p-1 + _p) * _n * node + (2*_p-1) * dim + cnt;
    _routers[node]->AddOutputChannel( _chan[_output], _chan_cred[_output] );
    
    #ifdef DRAGON_LATENCY
    _chan[_output]->SetLatency(10);
    _chan_cred[_output]->SetLatency(10);
    #endif
  }
}
```

### Step 2.4: Connecting Global Inter-Group Links ("Optical" Channels)
Global inter-group channels (optical links) connect switches in different groups. Under the `DRAGON_LATENCY` compilation macro, these long-distance links are configured with a hardcoded latency of **100 cycles**:
```cpp
for ( int cnt = 0; cnt < _p; ++cnt ) {
  _output = (2*_p-1 + _p) * node + (2*_p - 1) + cnt;
  _routers[node]->AddOutputChannel( _chan[_output], _chan_cred[_output] );
  
  #ifdef DRAGON_LATENCY
  _chan[_output]->SetLatency(100);
  _chan_cred[_output]->SetLatency(100);
  #endif
}
```
This hardcoded asymmetry (10 vs 100 cycles) is key to modeling the high-latency cost of global optical interconnects in Dragonfly networks.
