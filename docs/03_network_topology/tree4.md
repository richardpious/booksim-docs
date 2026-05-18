[<- Topology Index](README.md)

# 4-ary Tree (Tree4) Topology

The 4-ary Tree (`tree4`) is a specialized indirect network topology designed for exactly 64 terminal processor nodes. It is implemented in the [Tree4](../../booksim/src/networks/tree4.cpp#L55) class inside [tree4.cpp](../../booksim/src/networks/tree4.cpp).

---

## 1. Parameters & Radical Switch Distribution

-   **`k` (`_k`)**: Radix (fixed at `4`).
-   **`n` (`_n`)**: Height (fixed at `3`).

> [!WARNING]
> Tree4 enforces an assertion requiring `_k == 4 && _n == 3`.

### 1.1 Symmetrical Router Sizing (`_ComputeSize`)
-   **Level 0 (Root)**: 4 switches, radix $8 \times 8$ (8 descending links).
-   **Level 1 (Middle)**: 8 switches, radix $8 \times 8$ (4 ascending, 4 descending links).
-   **Level 2 (Leaf)**: 16 switches, radix $6 \times 6$ (2 ascending, 4 descending links).
-   **Level 3 (Terminal)**: 64 terminal processor nodes.
-   **Total Switches (`_size`)**: `4 + 8 + 16 = 28`
-   **Total Inter-router Channels (`_channels`)**: `128`

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/tree4.cpp#L89) method sets up switches and registers connections.

### 2.1 Router Sizing
Switch radix varies depending on hierarchical height $h$:
-   **Levels 0 and 1 (Roots/Middle)**: Sized with degree `8`.
-   **Level 2 (Leaf)**: Sized with degree `6` (only 2 ascending and 4 descending ports required).
```cpp
if ( h < _n-1 ) 
  degree = 8;
else
  degree = 6;
_Router( h, pos ) = Router::NewRouter( config, this, name.str( ), id, degree, degree );
```

### 2.2 Port Connections & Routing Directions
Connections between stages use a standard directional rule:
-   **Output ports $0:3$**: Move DOWN the tree (toward leaves).
-   **Output ports $4:7$**: Move UP the tree (toward roots).

### 2.3 Leaf concentration links (Bottom Stage)
Leaf routers at level 2 connect downward ports directly to terminal node injection and ejection channels. These boundary links default to **1 cycle** latency:
```cpp
for ( int port = 0 ; port < _k ; ++port ) {
  _Router( _n-1, pos)->AddInputChannel( _inject[_k*pos+port], _inject_cred[_k*pos+port]);
  _Router( _n-1, pos)->AddOutputChannel( _eject[_k*pos+port], _eject_cred[_k*pos+port]);
}
```

### 2.4 Middle (h=1) to Leaf (h=2) Wires
Middle routers connect to leaf switches resolved as `_k * (pos / 2) + port` (ports 0 to 3):
```cpp
pp = pos;
pc = _k * ( pos / 2 ) + port;
_Router( 1, pp)->AddOutputChannel( _chan[c], _chan_cred[c] );
_Router( 2, pc)->AddInputChannel(  _chan[c], _chan_cred[c] );
```

### 2.5 Root (h=0) to Middle (h=1) Wires
Root routers connect to middle switches resolved as `port` (ports 0 to 3):
```cpp
pp = pos;
pc = port;
_Router(0, pp)->AddOutputChannel( _chan[c], _chan_cred[c] );
_Router(1, pc)->AddInputChannel( _chan[c], _chan_cred[c] );
```

---

## 3. Asymmetric Wire Latency Calculation

Tree4 features a custom [_WireLatency](../../booksim/src/networks/tree4.cpp#L226) delay solver to model physical layout cabling constraints:
-   **Leaf-to-Middle Links (`height == 2`)**: Latency is fixed at **2 cycles** (`_length_d2_d1 = 2`).
-   **Middle-to-Root Links (`height == 1`)**: Latency varies between **2 and 6 cycles** depending on the parent-child coordinates:
    ```cpp
    int _length_d1_d0_0 = 2;
    int _length_d1_d0_1 = 2;
    int _length_d1_d0_2 = 6;
    int _length_d1_d0_3 = 6;
    ```
    This is evaluated using coordinate lookups:
    ```cpp
    if ( posChild == 0 || posChild == 6 )
      switch ( posParent ) {
        case 0: L = _length_d1_d0_0; break;
        case 2: L = _length_d1_d0_2; break;
      }
    ```
