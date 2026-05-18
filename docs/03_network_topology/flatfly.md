[<- Topology Index](README.md)

# Flattened Butterfly (FlatFly) In-Depth Guide

The Flattened Butterfly (`flatfly`) is a high-radix on-chip direct topology that connects switches in fully-connected cliques across multiple dimensions. It is implemented in the [FlatFlyOnChip](../../booksim/src/networks/flatfly_onchip.cpp#L68) class inside [flatfly_onchip.cpp](../../booksim/src/networks/flatfly_onchip.cpp).

---

## 1. Parameters & Radical Switch Sizing

The topology size is configured by the following variables:
-   `k` (`_k`): Number of routers along each dimension.
-   `n` (`_n`): Number of dimensions (supports up to 4 dimensions).
-   `c` (`_c`): Concentration degree (terminal nodes per router).
-   `x`, `y`: Router dimensions (must satisfy `x == y`).
-   `xr`, `yr`: Spatial grid coordinates of terminal nodes within a router (must satisfy `c = xr * yr` and `xr == yr`).

### Radix Calculation (`_ComputeSize`)
Unlike standard meshes, the total radix $r$ (number of ports per switch) grows with both dimension and router radix because it includes fully connected direct links to all other switches in the same dimension:
```cpp
_r             = _c + (_k - 1) * _n;  // Ports per switch
_nodes         = powi( _k, _n ) * _c; // Total processor nodes
_num_of_switch = _nodes / _c;         // Total switches
_channels      = _num_of_switch * (_r - _c);
```

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/flatfly_onchip.cpp#L106) method creates direct connections between all switches sharing a row, column, or higher-dimensional line.

```text
               Dimension Clique Concept (k=4)
      [Switch 0] <=========================> [Switch 1]
          ||  \                                /  ||
          ||   \                              /   ||
          ||    \                            /    ||
          ||     \                          /     ||
          \/      \                        /      \/
      [Switch 2] <=========================> [Switch 3]
      (Every switch has direct links to all others in its dimension)
```

### Step 2.1: Router Allocation
Routers are allocated with the compiled radix `_r`:
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, _r, _r );
```

### Step 2.2: Spatial Local Client Mapping & Delay
Clients are concentrated around each router in a localized grid. When `use_noc_latency = 1` is configured, the local wire length is modeled as an offset Manhattan distance (`ileng`) from the center of the router to the client node:
```cpp
// Coordinate delta estimation
int yleng = -_yrouter/2;
int xleng = -_xrouter/2;
// ...
int ileng = 1; // Base delay
if(abs(yleng)>1) ileng += (abs(yleng)-1);
if(abs(xleng)>1) ileng += (abs(xleng)-1);
```
These latencies are bound directly to the local injection/ejection channels:
```cpp
_inject[link]->SetLatency(ileng);
_eject[link]->SetLatency(ileng);
```

### Step 2.3: Inter-Router Dimension Clique Interconnections
For each router `node` and dimension `dim`, the switch connects to every other switch coordinate index (`cnt`) in that dimension:
1.  **Coordinate Mapping**: The builder maps multi-dimensional coordinates for up to 4 dimensions ($X, Y, Z, W$) to resolve neighbor indices:
    ```cpp
    int xcurr = node % _k;
    int ycurr = (int)(node / _k);
    int curr3 = node % (_k * _k);
    int curr4 = (int)(node / (_k * _k));
    // ...
    ```
2.  **Dimension Clique Binding**: Links are established between the current router and the `other` neighbor:
    ```cpp
    _output = (_k-1) * _n * node + (_k-1) * dim + cnt + offset;
    _routers[node]->AddOutputChannel( _chan[_output], _chan_cred[_output] );
    _routers[other]->AddInputChannel( _chan[_output], _chan_cred[_output] );
    ```
3.  **Physical Link Delay**: When `use_noc_latency = 1` is configured, the inter-router channel latency is calculated using the layout distance between switches:
    ```cpp
    int oned = abs((node % _xcount) - (other % _xcount));
    int twod = abs(node / _xcount - other / _xcount);
    int length = _xrouter * oned + _yrouter * twod;
    
    _chan[_output]->SetLatency(length);
    _chan_cred[_output]->SetLatency(length);
    ```
    This results in higher wire delay values for long-distance clique links, realistically modeling global wire performance on a chip.
