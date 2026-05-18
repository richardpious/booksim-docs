[<- Topology Index](README.md)

# Butterfly Stage (Fly) Topology

The Butterfly Stage (`fly`) is a classic multi-stage indirect network topology. It is implemented in the [KNFly](../../booksim/src/networks/fly.cpp#L37) class inside [fly.cpp](../../booksim/src/networks/fly.cpp).

---

## 1. Top-Level Sizing & Parameters

-   **`k` (`_k`)**: Radix (defining the uniform $k \times k$ switch crossbar size).
-   **`n` (`_n`)**: Dimension/stage count.

### 1.1 Sizing Formulas (`_ComputeSize`)
-   **Total Terminals (`_nodes`)**: `powi(_k, _n)`
-   **Total Switches (`_size`)**: `_n * powi(_k, _n - 1)`
-   **Total Inter-stage Channels (`_channels`)**:
    ```cpp
    _channels = (_n - 1) * _nodes;
    ```

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/fly.cpp#L61) method instantiates $k \times k$ switches stage-by-stage and registers inter-stage butterfly connections.

### 2.1 Switch Sizing
Switches are allocated with a uniform radix of `_k \times _k` at every stage:
```cpp
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, _k, _k );
```

### 2.2 Boundary Terminal Connections
Processor injection and ejection channels are wired only to boundary stages (0 and $n-1$), using a default latency of **1 cycle**:
-   **Injection (Input to Stage 0)**:
    ```cpp
    if ( stage == 0 ) {
      c = addr * _k + port;
      _routers[node]->AddInputChannel( _inject[c], _inject_cred[c] );
    }
    ```
-   **Ejection (Output from Stage $n-1$)**:
    ```cpp
    if ( stage == _n - 1 ) {
      c = addr * _k + port;
      _routers[node]->AddOutputChannel( _eject[c], _eject_cred[c] );
    }
    ```

### 2.3 Inter-Stage Channels (Outgoings)
For non-boundary stages ($stage < n-1$), outgoing channels are registered sequentially:
```cpp
c = _OutChannel( stage, addr, port );
_routers[node]->AddOutputChannel( _chan[c], _chan_cred[c] );
```
Where `_OutChannel` returns `stage * _nodes + addr * _k + port`.

### 2.4 Digit-Swapping Input Channels
For stages after the first ($stage > 0$), incoming channels map using coordinate digit-swapping arithmetic in `_InChannel` to form the butterfly routing fabric:
```cpp
int shift = powi( _k, _n-stage-1 );
int last_digit = port;
int zero_digit = ( addr / shift ) % _k;

// Swap coordinates to solve source address
in_addr = addr - zero_digit*shift + last_digit*shift;
in_port = zero_digit;

return (stage-1)*_nodes + in_addr*_k + in_port;
```
This digit swapping ensures that packets can be routed from any source terminal to any destination terminal in exactly $n$ stage traversals.
