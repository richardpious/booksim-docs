[<- Topology Index](README.md)

# Flattened Butterfly (FlatFly) Topology

The Flattened Butterfly (`flatfly`) is a high-radix on-chip direct topology that establishes fully-connected cliques of switches across multiple dimensions. It is implemented in the [FlatFlyOnChip](../../booksim/src/networks/flatfly_onchip.cpp#L68) class inside [flatfly_onchip.cpp](../../booksim/src/networks/flatfly_onchip.cpp).

---

## 1. Sizing Parameters & Radix Calculations

-   **`k` (`_k`)**: Switch dimension radix (switches along each dimension).
-   **`n` (`_n`)**: Number of dimensions (supports up to 4).
-   **`c` (`_c`)**: Concentration degree (terminal nodes per switch).
-   **`xcount`, `ycount`**: Symmetrical router grid width and height (must be equal).
-   **`xrouter`, `yrouter`**: Concentration layout grid coordinates (must satisfy `c = xrouter * yrouter` and `xrouter == yrouter`).

### 1.1 Structural Sizing Metrics (`_ComputeSize`)
-   **Switch Radix (`_r`)**:
    ```cpp
    _r = _c + (_k - 1) * _n;
    ```
-   **Total Terminals (`_nodes`)**: `powi(_k, _n) * _c`
-   **Total Switches (`_num_of_switch`)**: `_nodes / _c`
-   **Total Inter-router Channels (`_channels`)**: `_num_of_switch * (_r - _c)`

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/flatfly_onchip.cpp#L106) method allocates switches and wires fully-connected rows/columns.

### 2.1 Router Sizing
Switches are allocated with the computed radix `_r`:
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, _r, _r );
```

### 2.2 Local Terminal Mapping & Delay
Terminal client injection and ejection channels are connected to ports $0$ to $c - 1$. Under `use_noc_latency = 1`, the local wire delay is modeled as an offset Manhattan distance (`ileng`) from the physical center of the router to the concentrated client coordinate:
```cpp
int yleng = -_yrouter/2;
int xleng = -_xrouter/2;
// ...
int ileng = 1; // Base delay
if(abs(yleng)>1) ileng += (abs(yleng)-1);
if(abs(xleng)>1) ileng += (abs(xleng)-1);
```

### 2.3 Dimension Clique Interconnections
For each switch coordinate and dimension `dim`, the current router establishes direct unidirectional channels to all other `_k - 1` routers in the same dimension:
```cpp
_output = (_k-1) * _n * node + (_k-1) * dim + cnt + offset;
_routers[node]->AddOutputChannel( _chan[_output], _chan_cred[_output] );
_routers[other]->AddInputChannel( _chan[_output], _chan_cred[_output] );
```
-   **Physical Link Latency**: When `use_noc_latency = 1` is configured, inter-router link delay is calculated from switch distance offsets:
    ```cpp
    int oned = abs((node % _xcount) - (other % _xcount));
    int twod = abs(node / _xcount - other / _xcount);
    int length = _xrouter * oned + _yrouter * twod;
    ```
