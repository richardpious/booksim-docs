[<- Topology Index](README.md)

# Butterfly Stage (Fly) In-Depth Guide

The Butterfly Stage (`fly`) is a classic multi-stage indirect network topology. In BookSim, this is implemented in the [KNFly](../../booksim/src/networks/fly.cpp#L37) class inside [fly.cpp](../../booksim/src/networks/fly.cpp).

---

## 1. Top-Level Sizing & Parameters

The topology size is configured by the following variables:
-   `k` (`_k`): Radix (number of ports per switch, defining the $k \times k$ crossbar size).
-   `n` (`_n`): Dimension/stage count.

### Sizing Formulas (`_ComputeSize`)
-   **Total Nodes (`_nodes`)**: `powi(_k, _n)`
-   **Total Switches (`_size`)**: `_n * powi(_k, _n - 1)` (composed of `_n` stages with `powi(_k, _n - 1)` switches per stage).
-   **Total Inter-stage Channels (`_channels`)**:
    ```cpp
    _channels = (_n - 1) * _nodes
    ```

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/fly.cpp#L61) method instantiates $k \times k$ switches stage-by-stage and establishes butterfly routing channels between consecutive stages:

```text
                      Multi-Stage Butterfly Concept
           [Stage 0]          [Stage 1]          [Stage 2]
         +-----------+      +-----------+      +-----------+
    In =>|  Switch   |=====\|  Switch   |=====\|  Switch   |=> Out
         +-----------+     /|-----------+     /|-----------+
         +-----------+    / +-----------+    / +-----------+
    In =>|  Switch   |===/==|  Switch   |===/==|  Switch   |=> Out
         +-----------+      +-----------+      +-----------+
         (Direct digit-swapping maps connections between stages)
```

### Step 2.1: Switch Sizing
Switches at every stage are allocated with a uniform radix of `_k \times _k`:
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, _k, _k );
```

### Step 2.2: Terminal Node Connections
Terminal node injection and ejection channels are connected exclusively to the boundary stages of the network:
-   **Injection Channels (Input to Stage 0)**:
    ```cpp
    if ( stage == 0 ) {
      c = addr * _k + port;
      _routers[node]->AddInputChannel( _inject[c], _inject_cred[c] );
    }
    ```
-   **Ejection Channels (Output from Stage $n-1$)**:
    ```cpp
    if ( stage == _n - 1 ) {
      c = addr * _k + port;
      _routers[node]->AddOutputChannel( _eject[c], _eject_cred[c] );
    }
    ```
-   These boundary terminal links are configured with a latency of **1 cycle**.

### Step 2.3: Inter-Stage Output Channels
For all non-boundary stages ($stage < n-1$), outgoing channels are mapped sequentially via `_OutChannel`:
```cpp
c = _OutChannel( stage, addr, port );
_routers[node]->AddOutputChannel( _chan[c], _chan_cred[c] );
```
Where `_OutChannel` returns: `stage * _nodes + addr * _k + port`.

### Step 2.4: Digit-Swapping Input Channels (The Core Butterfly Map)
For stages after the first ($stage > 0$), incoming channels are mapped to input ports using a custom **Digit-Swapping** arithmetic inside `_InChannel`. This solver swaps coordinates of the switch address to route flits through the butterfly structure:
```cpp
int shift = powi( _k, _n-stage-1 );
int last_digit = port;
int zero_digit = ( addr / shift ) % _k;

// Swap zero and last digit to solve source node address
in_addr = addr - zero_digit*shift + last_digit*shift;
in_port = zero_digit;

return (stage-1)*_nodes + in_addr*_k + in_port;
```
This digit swapping ensures that packets can be routed from any source terminal to any destination terminal in exactly $n$ hops, which is the defining characteristic of butterfly multi-stage interconnection networks (MINs).
