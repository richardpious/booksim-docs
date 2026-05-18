[<- Topology Index](README.md)

# Fat-Tree Topology

The Fat-Tree is a hierarchical multi-stage indirect network that provides high bisection bandwidth. It is implemented in the [FatTree](../../booksim/src/networks/fattree.cpp#L58) class inside [fattree.cpp](../../booksim/src/networks/fattree.cpp).

---

## 1. Parameters & Tree Sizing

-   **`k` (`_k`)**: Radix (number of descending child links per router).
-   **`n` (`_n`)**: Level count (tree height).

### 1.1 Sizing Formulas (`_ComputeSize`)
-   **Total Terminals (`_nodes`)**: `powi(_k, _n)`
-   **Total Switches (`_size`)**: `_n * powi(_k, _n - 1)`
-   **Total Inter-router Channels (`_channels`)**:
    ```cpp
    _channels = (2 * _k * powi(_k, _n - 1)) * (_n - 1);
    ```

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/fattree.cpp#L93) method allocates switches level-by-level and connects hierarchical parent-child stages.

### 2.1 Switch Allocation by Level
Switches are organized in levels from `0` (root) to `_n - 1` (leaves). Router degrees differ depending on their position in the hierarchy:
-   **Root Switches (`level == 0`)**: Sized with degree `_k` (only downward links).
-   **Middle & Leaf Switches (`level > 0`)**: Sized with degree `2 * _k` (supporting `_k` upward and `_k` downward links).
```cpp
_routers[id] = Router::NewRouter( config, this, name.str( ), id, degree, degree );
```

### 2.2 Directional Port Rules
Ports are partitioned according to their physical routing direction:
-   **Output ports $< k$**: Move DOWN the tree (toward leaves).
-   **Output ports $\ge k$**: Move UP the tree (toward roots).
-   **Input ports $< k$**: Receive flits from switches DOWN the tree.
-   **Input ports $\ge k$**: Receive flits from switches UP the tree.

### 2.3 Leaf concentration links (Bottom Stage)
The leaf routers (level `_n - 1`) connect downward-facing ports $0$ to $k-1$ directly to processor injection and ejection channels. These boundary links have a default latency of **1 cycle**:
```cpp
_Router( _n-1, pos)->AddInputChannel( _inject[link], _inject_cred[link] );
_Router( _n-1, pos)->AddOutputChannel( _eject[link], _eject_cred[link] );
```

### 2.4 Middle Stage Inter-Router Wires
Inter-stage outgoing channels are registered sequentially:
-   **Downward Outputs** (levels $0$ to $n-2$):
    ```cpp
    link = (level * chan_per_level) + pos * _k + port;
    ```
-   **Upward Outputs** (levels $1$ to $n-1$):
    ```cpp
    link = (level * chan_per_level - chan_per_direction) + pos * _k + port;
    ```

### 2.5 Mathematical Branch Interleaving
Downward and upward input channels are mapped to ports using a mathematical interleaving step to resolve parent-child branches:
-   **Downward Inputs**:
    ```cpp
    int routers_per_neighborhood = powi(_k, _n-1-level); 
    int routers_per_branch = powi(_k, _n-1-(level+1)); 
    int level_offset = routers_per_neighborhood * _k;
    int link = ((level+1) * chan_per_level - chan_per_direction) + neighborhood * level_offset + port * routers_per_branch * gK + (neighborhood_pos) % routers_per_branch * gK + (neighborhood_pos) / routers_per_branch;
    ```
