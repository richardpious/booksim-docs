[<- Topology Index](README.md)

# Quad Tree (QTree) Topology

The Quad Tree is a hierarchical indirect network featuring 4 descending links per parent router. It is implemented in the [QTree](../../booksim/src/networks/qtree.cpp#L48) class inside [qtree.cpp](../../booksim/src/networks/qtree.cpp).

---

## 1. Parameter Constraints & Sizing

-   **`k` (`_k`)**: Radix (fixed at `4`).
-   **`n` (`_n`)**: Height (fixed at `3`).

> [!WARNING]
> QTree enforces a hardcoded assertion requiring `_k == 4 && _n == 3`.

### 1.1 Structural Sizing Formulas (`_ComputeSize`)
-   **Total Terminals (`_nodes`)**: `powi(_k, _n) = 64`
-   **Total Switches (`_size`)**:
    ```cpp
    _size = sum_{i=0}^{n-1} k^i = 1 + 4 + 16 = 21
    ```
-   **Total Inter-router Channels (`_channels`)**:
    ```cpp
    _channels = sum_{j=1}^{n-1} 2 * k^j = 2 * (4 + 16) = 40
    ```

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/qtree.cpp#L83) method instantiates switches and establishes parent-child connections.

### 2.1 Switch Allocation by Layer
Switches are allocated stage-by-stage. Switch degree corresponds to height $h$ in the tree:
-   **Root Switch (`h == 0`)**: Sized with degree `_k = 4` (no parent link).
-   **Other Switches (`h > 0`)**: Sized with degree `_k + 1 = 5` (supporting 4 children and 1 parent).
```cpp
int d = ( h == 0 ) ? _k : _k + 1;
_routers[r] = Router::NewRouter( config, this, routerName.str( ), id, d, d );
```

### 2.2 Leaf Stage Concentration Wires
Switches at the leaf stage ($h = 2$) connect downward ports directly to terminal node injection and ejection channels. These links have a default latency of **1 cycle**:
```cpp
int r = _RouterIndex( _n-1, pos );
for ( int port = 0 ; port < _k ; port++ ) {
  _routers[r]->AddInputChannel( _inject[_k*pos+port], _inject_cred[_k*pos+port] );
  _routers[r]->AddOutputChannel( _eject[_k*pos+port], _eject_cred[_k*pos+port] );
}
```

### 2.3 Downward Child Channels (Levels 0 and 1)
For non-leaf switches ($h < 2$), downward child channels are registered via `_InputIndex` and `_OutputIndex`:
```cpp
c = _InputIndex( h , pos, port );
_routers[r]->AddInputChannel( _chan[c], _chan_cred[c] );

c = _OutputIndex( h, pos, port );
_routers[r]->AddOutputChannel( _chan[c], _chan_cred[c] );
```

### 2.4 Upward Parent Channels (Levels 1 and 2)
For switches below the root ($h > 0$), upward channels connect to parent switches. The parent switch index is solved as `pos / _k` (integer division) and the port index maps to `pos % _k`:
```cpp
c = _OutputIndex( h - 1, pos / _k, pos % _k );
_routers[r]->AddInputChannel( _chan[c], _chan_cred[c] );

c = _InputIndex( h - 1, pos / _k, pos % _k );
_routers[r]->AddOutputChannel( _chan[c], _chan_cred[c] );
```
