[<- Topology Index](README.md)

# Quad Tree (QTree) In-Depth Guide

The Quad Tree (`qtree`) is a hierarchical indirect network featuring 4 descending links per parent router. In BookSim, this is implemented in the [QTree](../../booksim/src/networks/qtree.cpp#L48) class inside [qtree.cpp](../../booksim/src/networks/qtree.cpp).

---

## 1. Parameters & Tree Sizing

The Quad Tree topology is configured by two variables but enforces a strict size constraint:
-   `k` (`_k`): Radix (fixed at `4`).
-   `n` (`_n`): Height (fixed at `3`).

> [!WARNING]
> QTree enforces an assertion that `_k == 4 && _n == 3`. Attempting to configure other values will result in a compilation/assertion crash.

### Sizing Formulas (`_ComputeSize`)
-   **Total Nodes (`_nodes`)**: `powi(_k, _n) = 64`
-   **Total Switches (`_size`)**:
    ```cpp
    _size = sum_{i=0}^{n-1} k^i = 1 + 4 + 16 = 21
    ```
-   **Total Inter-Router Channels (`_channels`)**:
    ```cpp
    _channels = sum_{j=1}^{n-1} 2 * k^j = 2 * (4 + 16) = 40
    ```

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/qtree.cpp#L83) method sets up the switches layer-by-layer:

```text
                        Quad Tree Network Layout
                        [Height 0: Root] (Degree 4)
                       //     ||       ||     \\
                    [Height 1: 4 Switches] (Degree 5)
                      //      ||       ||      \\
                   [Height 2: 16 Switches] (Degree 5)
                     //       ||       ||       \\
                [64 Terminal Processor Nodes] (Injection/Ejection)
```

### Step 2.1: Router Allocation by Layer
Switches are allocated layer-by-layer. Since the topmost root router ($h=0$) has no upward parent interface, its degree is smaller than the mid/bottom switches:
-   **Root Switch (`h == 0`)**: Sized with degree `_k = 4` (only downwards child links).
-   **Other Switches (`h > 0`)**: Sized with degree `_k + 1 = 5` (4 child links downwards plus 1 parent link upwards).
```cpp
int d = ( h == 0 ) ? _k : _k + 1;
_routers[r] = Router::NewRouter( config, this, routerName.str( ), id, d, d );
```

### Step 2.2: Injection and Ejection Binding (Bottom Stage)
The switches at the leaf stage ($h = \text{height} - 1 = 2$) connect their child interfaces directly to the terminal node injection/ejection channels, configured with a latency of **1 cycle**:
```cpp
int r = _RouterIndex( _n-1, pos );
for ( int port = 0 ; port < _k ; port++ ) {
  _routers[r]->AddInputChannel( _inject[_k*pos+port], _inject_cred[_k*pos+port] );
  _routers[r]->AddOutputChannel( _eject[_k*pos+port], _eject_cred[_k*pos+port] );
}
```

### Step 2.3: Child Channels Wiring (Levels 0 and 1)
For non-leaf switches ($h < 2$), the builder wires downwards channels to children using `_InputIndex` and `_OutputIndex`:
```cpp
c = _InputIndex( h , pos, port );
_routers[r]->AddInputChannel( _chan[c], _chan_cred[c] );

c = _OutputIndex( h, pos, port );
_routers[r]->AddOutputChannel( _chan[c], _chan_cred[c] );
```

### Step 2.4: Parent Channels Wiring (Levels 1 and 2)
For switches below the root stage ($h > 0$), the builder wires upward channels to their designated parent switch. The parent index is solved as `pos / _k` (integer division) and the port index is mapped to `pos % _k`:
```cpp
c = _OutputIndex( h - 1, pos / _k, pos % _k );
_routers[r]->AddInputChannel( _chan[c], _chan_cred[c] );

c = _InputIndex( h - 1, pos / _k, pos % _k );
_routers[r]->AddOutputChannel( _chan[c], _chan_cred[c] );
```
This direct `pos / _k` mapping clusters 4 child switches onto each parent router stage by stage, resulting in the quad-tree structure.
