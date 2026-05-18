[<- Topology Index](README.md)

# Concentrated Mesh (CMesh) Topology

The Concentrated Mesh clusters multiple terminal nodes onto a single shared router. It is implemented in the [CMesh](../../booksim/src/networks/cmesh.cpp#L56) class inside [cmesh.cpp](../../booksim/src/networks/cmesh.cpp).

---

## 1. Parameters & Structural Constraints

-   **`k` (`_k`)**: Radix (grid width).
-   **`n` (`_n`)**: Dimensions (must be $\le 2$).
-   **`c` (`_c`)**: Concentration factor (must be $4$).
-   **`xcount`, `ycount`**: Symmetrical router grid width and height (must be equal).
-   **`xrouter`, `yrouter`**: Symmetrical concentration width and height per switch (must satisfy `xrouter * yrouter = c` and `xrouter == yrouter`).

### 1.1 Sizing Formulas (`_ComputeSize`)
-   **Total Switches (`_size`)**: `powi(_k, _n)`
-   **Total Inter-router Channels (`_channels`)**: `2 * _n * _size`
-   **Total Nodes (`_nodes`)**: `_c * _size`
-   **Dimensional Concentration Factors**:
    ```cpp
    _cX = _c / _n;     // Concentration in X
    _cY = _c / _cX;    // Concentration in Y
    ```

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/cmesh.cpp#L109) method constructs routers and wires both concentration and inter-router links.

### 2.1 Router Port Mapping Matrix
Switches are allocated with degree $2n + c$ (ports $0$ to $2n+c-1$). Ports are allocated in the following strict order:
-   **Ports $0$ to $c-1$ ($0:3$)**: Concentrated processor nodes.
-   **Port $c + 0$ ($4$)**: Positive X (+X) neighboring router.
-   **Port $c + 1$ ($5$)**: Negative X (-X) neighboring router.
-   **Port $c + 2$ ($6$)**: Positive Y (+Y) neighboring router.
-   **Port $c + 3$ ($7$)**: Negative Y (-Y) neighboring router.

### 2.2 Terminal Node Connections
The builder maps injection and ejection channels for the concentrated terminal nodes to ports $0$ to $c-1$:
```cpp
int link = (_k * _cX) * (_cY * y_index + y) + (_cX * x_index + x);
_routers[node]->AddInputChannel(_inject[link], _inject_cred[link]);
_routers[node]->AddOutputChannel(_eject[link], _eject_cred[link]);
```
*Note:* Concentration links always default to a standard latency of **1 cycle**.

### 2.3 Express Boundary Channels
To reduce network diameter, edge switches route boundary traffic to opposite-side switches half-grid distance away vertically/horizontally:
-   **Left Edge (`x == 0`)**: The negative X input `nx_in` connects to a router offset by $\frac{k}{2}$ steps vertically.
-   **Right Edge (`x == _k - 1`)**: The positive X input `px_in` connects to a router offset by $\frac{k}{2}$ steps vertically.
-   **Bottom Edge (`y == 0`)**: The negative Y input `ny_in` connects to a router offset by $\frac{k}{2}$ steps horizontally.
-   **Top Edge (`y == _k - 1`)**: The positive Y input `py_in` connects to a router offset by $\frac{k}{2}$ steps horizontally.

### 2.4 Wire Delay Modeling (`use_noc_latency`)
When `use_noc_latency = 1` is configured, express links scale their latency based on physical Manhattan coordinate distance, while standard links remain single-cycle:
-   **Positive X (+X) Latency**: `(x == _k - 1) ? (_cY * _k / 2) : _cX;`
-   **Positive Y (+Y) Latency**: `(y == _k - 1) ? (_cX * _k / 2) : _cY;`
