[<- Topology Index](README.md)

# Dragonfly Topology

The Dragonfly hierarchical topology features tightly-connected groups of switches joined by long global channels. It is implemented in the [DragonFlyNew](../../booksim/src/networks/dragonfly.cpp#L149) class inside [dragonfly.cpp](../../booksim/src/networks/dragonfly.cpp).

> [!IMPORTANT]
> To use this topology, configure `topology = dragonflynew` in your configuration file.

---

## 1. Parameters & Group Architecture

-   **`k` (`_p`)**: Switch local radix (defines the number of concentrated terminal ports per switch).
-   **`n` (`_n`)**: Group dimension (must be set to `1`).

### 1.1 Structural Sizing Metrics (`_ComputeSize`)
-   **Routers per Group (`_a`)**: `2 * _p`
-   **Group Count (`_g`)**: `_a * _p + 1`
-   **Total Terminals (`_nodes`)**: `_a * _p * _g`
-   **Total Switches (`_num_of_switch`)**: `_nodes / _p`
-   **Total Inter-router Channels (`_channels`)**: `_num_of_switch * (_k - _p)`
-   **Switch Total Radix (`_k`)**:
    ```cpp
    _k = _p + _p + 2 * _p - 1;
    ```
    The port allocation consists of:
    *   `_p` local terminal ports.
    *   `_p` global inter-group ports.
    *   `2 * _p - 1` local intra-group inter-switch ports.

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/dragonfly.cpp#L215) method allocates switches and wires three distinct classes of links.

### 2.1 Switch Allocation
Switches are allocated with the computed radix `_k`:
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, _k, _k );
```

### 2.2 Terminal Connections
Processor nodes connect to the first `_p` ports of each router. These injection/ejection channels are assigned a default latency of **1 cycle**:
```cpp
c = _p * node + cnt;
_routers[node]->AddInputChannel( _inject[c], _inject_cred[c] );
_routers[node]->AddOutputChannel( _eject[c], _eject_cred[c] );
```

### 2.3 Intra-Group Inter-Router Links
Each router connects to the other `2 * _p - 1` routers within its local group. When the `DRAGON_LATENCY` compilation macro is defined, these intra-group links are assigned a hardcoded latency of **10 cycles**:
```cpp
_output = (2*_p-1 + _p) * _n * node + (2*_p-1) * dim + cnt;
_routers[node]->AddOutputChannel( _chan[_output], _chan_cred[_output] );
#ifdef DRAGON_LATENCY
_chan[_output]->SetLatency(10);
_chan_cred[_output]->SetLatency(10);
#endif
```

### 2.4 Global Inter-Group Links (Optical Channels)
Global inter-group channels connect switches in different groups. When the `DRAGON_LATENCY` compilation macro is defined, these long-distance links are assigned a hardcoded latency of **100 cycles**:
```cpp
_output = (2*_p-1 + _p) * node + (2*_p - 1) + cnt;
_routers[node]->AddOutputChannel( _chan[_output], _chan_cred[_output] );
#ifdef DRAGON_LATENCY
_chan[_output]->SetLatency(100);
_chan_cred[_output]->SetLatency(100);
#endif
```
