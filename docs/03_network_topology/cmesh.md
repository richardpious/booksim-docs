[<- Topology Index](README.md)

# Concentrated Mesh (CMesh) In-Depth Guide

The Concentrated Mesh (CMesh) topology clusters multiple terminal processor nodes onto a single shared router. It is implemented in the [CMesh](../../booksim/src/networks/cmesh.cpp#L56) class inside [cmesh.cpp](../../booksim/src/networks/cmesh.cpp).

---

## 1. Parameters & Configuration Constraints

The `_ComputeSize` method extracts grid metrics and enforces system constraints:
-   `k` (`_k`): Radix (router grid width).
-   `n` (`_n`): Dimensions (must satisfy `n <= 2`).
-   `c` (`_c`): Concentration degree (must satisfy `c == 4`).
-   `x`, `y`: Router grid dimensions (must satisfy `x == y`).
-   `xr`, `yr`: Terminal node concentration layout per router (must satisfy `c = xr * yr` and `xr == yr`).

### Grid Mappings
The concentration factors are decomposed in X and Y dimensions:
```cpp
_cX = _c / _n;     // Concentration along the X dimension (e.g., 2)
_cY = _c / _cX;    // Concentration along the Y dimension (e.g., 2)
```

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/cmesh.cpp#L109) method executes the network layout, assigning ports in a strict sequence:

```text
       Router Port Assignment Matrix
   +------------------------------------+
   |  Ports 0 to c-1 (0:3):             | --> Attached processor nodes
   |  Port c + 0     (4):               | --> +X neighboring router
   |  Port c + 1     (5):               | --> -X neighboring router
   |  Port c + 2     (6):               | --> +Y neighboring router
   |  Port c + 3     (7):               | --> -Y neighboring router
   +------------------------------------+
```

### Step 2.1: Mapping Concentrated Processor Nodes
For each router node at coordinates `(x_index, y_index)`, the builder maps the injection and ejection channels of the concentrated processor nodes onto ports $0$ to $c - 1$:
```cpp
for (int y = 0; y < _cY ; y++) {
  for (int x = 0; x < _cX ; x++) {
    int link = (_k * _cX) * (_cY * y_index + y) + (_cX * x_index + x);
    _routers[node]->AddInputChannel(_inject[link], _inject_cred[link]);
    _routers[node]->AddOutputChannel(_eject[link], _eject_cred[link]);
  }
}
```
*Note:* Concentration links always default to a standard latency of **1 cycle**.

### Step 2.2: Middle Router Interconnections
To calculate standard inter-router channel index offsets (ports $c$ through $c + 3$), the builder assigns unidirectional positive and negative connections:
-   `px_out` (+X Output): `node + 0 * offset`
-   `nx_out` (-X Output): `node + 1 * offset`
-   `py_out` (+Y Output): `node + 2 * offset`
-   `ny_out` (-Y Output): `node + 3 * offset`

Where `offset = powi(_k, _n)` represents the router count.

### Step 2.3: Express Boundary Channels
To reduce overall network diameter and bypass multi-hop routes across grid borders, CMesh implements specialized **Express Channels** on edge routers. These links route packet traffic across half-grid distances:

*   **Left Edge (`x == 0`)**: The negative X input `nx_in` connects to a router offset by $\frac{k}{2}$ steps vertically:
    ```cpp
    if (y < _k / 2)
      nx_in = _k * (y + _k/2) + x + offset;
    else
      nx_in = _k * (y - _k/2) + x + offset;
    ```
*   **Right Edge (`x == _k - 1`)**: The positive X input `px_in` connects to a router offset by $\frac{k}{2}$ steps vertically.
*   **Bottom Edge (`y == 0`)**: The negative Y input `ny_in` connects to a router offset by $\frac{k}{2}$ steps horizontally.
*   **Top Edge (`y == _k - 1`)**: The positive Y input `py_in` connects to a router offset by $\frac{k}{2}$ steps horizontally.

### Step 2.4: Spatial Delay Calculation (`use_noc_latency`)
When coordinate-based delays are enabled, express links scale their latency to reflect physical wire distances. Standard links remain single-cycle:
-   **Positive X (+X) Latency**:
    ```cpp
    int const px_latency = (x == _k-1) ? (_cY * _k / 2) : _cX;
    ```
-   **Positive Y (+Y) Latency**:
    ```cpp
    int const py_latency = (y == _k-1) ? (_cX * _k / 2) : _cY;
    ```
-   This ensures that the simulated clock cycle costs of crossing long express boundary lines are modeled realistically.
