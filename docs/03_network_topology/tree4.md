[<- Topology Index](README.md)

# 4-ary Tree (Tree4) In-Depth Guide

The 4-ary Tree (`tree4`) is a specialized indirect network topology designed for exactly 64 terminal processor nodes. In BookSim, this is implemented in the [Tree4](../../booksim/src/networks/tree4.cpp#L55) class inside [tree4.cpp](../../booksim/src/networks/tree4.cpp).

---

## 1. Symmetrical Top-Level Structure

The topology size is configured by the following variables and enforces a strict parameter assertion:
-   `k` (`_k`): Radix (fixed at `4`).
-   `n` (`_n`): Height (fixed at `3`).

> [!WARNING]
> Tree4 enforces an assertion that `_k == 4 && _n == 3`. Attempting to configure other values will result in a crash.

### Hierarchical Router Distribution (`_ComputeSize`)
-   **Level 0 (Root)**: 4 switches of radix $8 \times 8$ (8 descending links).
-   **Level 1 (Middle)**: 8 switches of radix $8 \times 8$ (4 ascending and 4 descending links).
-   **Level 2 (Leaf)**: 16 switches of radix $6 \times 6$ (2 ascending and 4 descending links).
-   **Level 3 (Terminal)**: 64 processor nodes.
-   **Total Switches (`_size`)**: `4 + 8 + 16 = 28` routers.
-   **Total Inter-Router Channels (`_channels`)**: `2 * (2 * powi(4, 1)) * (2 * 4) = 128` channels.

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/tree4.cpp#L89) method sets up the switches and wires:

```text
                        Tree4 Symmetrical Layout
               [Level 0: 4 Root Routers] (Degree 8)
                         /            \
             [Level 1: 8 Middle Routers] (Degree 8)
                       /                \
             [Level 2: 16 Leaf Routers] (Degree 6)
                     //   \\        //   \\
                 [64 Terminal Processor Nodes] (Injection/Ejection)
```

### Step 2.1: Router Sizing
Switches are allocated level-by-level, and their ports reflect their position in the hierarchy:
-   **Levels 0 and 1 (Roots/Middle)**: Sized with degree `8`.
-   **Level 2 (Leaf)**: Sized with degree `6` (only 2 ascending and 4 descending ports needed).
```cpp
if ( h < _n-1 ) 
  degree = 8;
else
  degree = 6;
_Router( h, pos ) = Router::NewRouter( config, this, name.str( ), id, degree, degree );
```

### Step 2.2: Port Connection Rules
Connections between stages adhere to a standard directional rule:
-   **Output ports $0:3$**: Move DOWN the network (toward leaves).
-   **Output ports $4:7$**: Move UP the network (toward roots).

### Step 2.3: Node Injection and Ejection (Bottom Level)
The 16 leaf routers (level 2) connect their 4 downward-facing ports directly to the 64 terminal node injection/ejection channels:
```cpp
for ( int port = 0 ; port < _k ; ++port ) {
  _Router( _n-1, pos)->AddInputChannel( _inject[_k*pos+port], _inject_cred[_k*pos+port]);
  _Router( _n-1, pos)->AddOutputChannel( _eject[_k*pos+port], _eject_cred[_k*pos+port]);
}
```

### Step 2.4: Connections Between Middle (h=1) and Leaf (h=2) Stages
Each middle router `pos` connects to 4 leaf switches resolved as `_k * (pos / 2) + port` (ports $0$ to $3$):
```cpp
pp = pos;
pc = _k * ( pos / 2 ) + port;
_Router( 1, pp)->AddOutputChannel( _chan[c], _chan_cred[c] );
_Router( 2, pc)->AddInputChannel(  _chan[c], _chan_cred[c] );
```

### Step 2.5: Connections Between Root (h=0) and Middle (h=1) Stages
Root routers at level 0 connect to middle switches at level 1:
```cpp
pp = pos;
pc = port;
_Router(0, pp)->AddOutputChannel( _chan[c], _chan_cred[c] );
_Router(1, pc)->AddInputChannel( _chan[c], _chan_cred[c] );
```

---

## 3. Asymmetric Wire Latency Calculation

Unlike regular trees that default to single-cycle hops, Tree4 implements a custom [_WireLatency](../../booksim/src/networks/tree4.cpp#L226) delay solver to model long physical cabling constraints inside layout boards:
-   **Leaf-to-Middle Links (`height == 2`)**: Latency defaults to **2 cycles** (`_length_d2_d1 = 2`).
-   **Middle-to-Root Links (`height == 1`)**: Latency varies between **2 and 6 cycles** based on coordinates (`_length_d1_d0_x` variables) to model variable grid lengths across root stages:
    ```cpp
    int _length_d1_d0_0 = 2;
    int _length_d1_d0_1 = 2;
    int _length_d1_d0_2 = 6;
    int _length_d1_d0_3 = 6;
    ```
    The solver maps these values in a custom switch matrix matching child and parent positions:
    ```cpp
    if ( posChild == 0 || posChild == 6 )
      switch ( posParent ) {
        case 0: L = _length_d1_d0_0; break; // 2 cycles
        case 2: L = _length_d1_d0_2; break; // 6 cycles
      }
    ```
