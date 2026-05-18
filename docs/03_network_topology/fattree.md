[<- Topology Index](README.md)

# Fat-Tree In-Depth Topology Guide

The Fat-Tree hierarchical indirect network features multiple stages of switches designed to provide high bisection bandwidth. In BookSim, this is implemented in the [FatTree](../../booksim/src/networks/fattree.cpp#L58) class inside [fattree.cpp](../../booksim/src/networks/fattree.cpp).

---

## 1. Grid Sizing & Parameters

The topology size is configured by two variables:
-   `k` (`_k`): Radix (number of child links per router).
-   `n` (`_n`): Levels/height of the tree.

### Sizing Formulas (`_ComputeSize`)
-   **Total Nodes (`_nodes`)**: `powi(_k, _n)`
-   **Total Switches (`_size`)**: `_n * powi(_k, _n - 1)` (composed of `_n` levels with `powi(_k, _n - 1)` switches per level).
-   **Total Inter-Router Channels (`_channels`)**:
    ```cpp
    _channels = (2 * _k * powi( _k , _n - 1 )) * (_n - 1);
    ```

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/fattree.cpp#L93) method builds the hierarchical tree structure layer-by-layer:

```text
                     Fat-Tree Hierarchical Structure
                         [Level 0: Root] (Degree k)
                             //        \\
                       [Level 1: Middle] (Degree 2k)
                       //              \\
                     [Level 2: Leaf] (Degree 2k)
                     //   \\          //   \\
                 [Node 0] [Node 1] [Node 2] [Node 3] (Processor Nodes)
```

### Step 2.1: Switch Allocation by Level
The builder loops through all stages `level` from `0` to `_n - 1` to allocate switches. The topmost root level has no upward parent links, meaning its router degree is smaller than intermediate/bottom levels:
-   **Root Switches (`level == 0`)**: Sized with degree `_k`.
-   **Other Switches (`level > 0`)**: Sized with degree `2 * _k` (to support `_k` downward and `_k` upward ports).
```cpp
_routers[id] = Router::NewRouter( config, this, name.str( ), id, degree, degree );
```

### Step 2.2: Port Connection Rules
The builder maps input and output channels to switches based on a strict spatial routing rule:
-   **Output ports $< k$**: Move DOWN the tree (toward the leaf level).
-   **Output ports $\ge k$**: Move UP the tree (toward the root level).
-   **Input ports $< k$**: Receive flits from switches DOWN the tree.
-   **Input ports $\ge k$**: Receive flits from switches UP the tree.

### Step 2.3: Leaf concentration links (Bottom Stage)
The leaf routers (level `_n - 1`) connect their downward-facing ports ($0$ to $k-1$) directly to the terminal node injection/ejection channels, configured with a latency of **1 cycle**:
```cpp
_Router( _n-1, pos)->AddInputChannel( _inject[link], _inject_cred[link] );
_Router( _n-1, pos)->AddOutputChannel( _eject[link], _eject_cred[link] );
```

### Step 2.4: Middle Stage Inter-Router Interconnections
The builder binds up and down channels to mid-level switches sequentially:
-   **Downward Outputs** (levels $0$ to $n-2$): Sized as `link = (level * chan_per_level) + pos * _k + port`.
-   **Upward Outputs** (levels $1$ to $n-1$): Sized as `link = (level * chan_per_level - chan_per_direction) + pos * _k + port`.

### Step 2.5: Interleaved Input Bindings
To connect branches properly and avoid network structural conflicts, inputs are bound to ports using interleaving formulas. The interleaving delta varies based on the current level:
-   **Downward Inputs**:
    ```cpp
    int routers_per_neighborhood = powi(_k, _n-1-level); 
    int routers_per_branch = powi(_k, _n-1-(level+1)); 
    int level_offset = routers_per_neighborhood * _k;
    int link = ((level+1) * chan_per_level - chan_per_direction) + neighborhood * level_offset + port * routers_per_branch * gK + (neighborhood_pos) % routers_per_branch * gK + (neighborhood_pos) / routers_per_branch;
    ```
-   **Upward Inputs**: Wired similarly using adjacent level offsets. This mathematical interleaving creates the high-bisection routing fabric characteristic of Fat-Trees.
