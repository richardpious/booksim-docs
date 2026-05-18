[<- Index](../index.md)

# Network Topology

BookSim supports a wide range of network topologies, ranging from traditional regular grids to high-radix on-chip layouts and custom, arbitrary graphs. The topology is specified using the `topology` parameter in your configuration file.

---

## 1. Available Topologies & Configuration

Here is the complete registry of supported topologies implemented in BookSim's source code, including spatial configurations and design parameters:

### Regular Grid & Mesh Topologies

*   **Mesh / Torus ([kncube.cpp](../../booksim/src/networks/kncube.cpp#L45))**:
    *   `topology = mesh` or `topology = torus`
    *   `k`: Radix (number of nodes per dimension).
    *   `n`: Dimension count.
    *   `c`: Concentration (nodes per router; default is 1).
    *   *Example*: `topology = mesh; k = 8; n = 2;` defines an 8x8 mesh grid.
*   **Concentrated Mesh ([cmesh.cpp](../../booksim/src/networks/cmesh.cpp#L56))**:
    *   `topology = cmesh`
    *   Optimized on-chip grid layout with multiple nodes clustered per router.
    *   `x`, `y`: Define the dimensions of the router grid.
    *   `xr`, `yr`: Define concentration grid dimensions per router (must satisfy concentration `c = xr * yr`).

### High-Radix On-Chip Topologies

*   **Flattened Butterfly ([flatfly_onchip.cpp](../../booksim/src/networks/flatfly_onchip.cpp#L68))**:
    *   `topology = flatfly`
    *   A high-radix, low-latency multi-dimensional topology designed for direct on-chip links.
    *   `k`: Number of routers per dimension.
    *   `n`: Number of dimensions.
    *   `c`: Concentration (nodes per router).
    *   `x`, `y`: Router dimensions (must satisfy `x == y`).
    *   `xr`, `yr`: Node arrangement grid per router (must satisfy `c = xr * yr`).
*   **Dragonfly ([dragonfly.cpp](../../booksim/src/networks/dragonfly.cpp#L149))**:
    *   `topology = dragonflynew`
    *   A modern high-radix hierarchical topology designed with tightly connected groups of switches joined by global links.
    *   `k`: Switch local radix (determines local ports `_p`).
    *   `n`: Group dimension (currently only `n = 1` is supported).
    *   > [!IMPORTANT]
    *   > Always specify `topology = dragonflynew` in the configuration. Specifying `topology = dragonfly` is an unrecognized identifier in BookSim.

### Tree & Stage Topologies

*   **Fat-Tree ([fattree.cpp](../../booksim/src/networks/fattree.cpp#L58))**:
    *   `topology = fattree`
    *   Hierarchical tree structure. Parameters `k` (radix) and `n` (height).
*   **Quad Tree ([qtree.cpp](../../booksim/src/networks/qtree.cpp#L96))**:
    *   `topology = qtree`
    *   Hierarchical tree structure where each parent router routes to up to 4 children routers or nodes.
*   **4-ary Tree ([tree4.cpp](../../booksim/src/networks/tree4.cpp#L99))**:
    *   `topology = tree4`
    *   A 4-ary tree network topology configuration.
*   **Butterfly Stage ([fly.cpp](../../booksim/src/networks/fly.cpp#L37))**:
    *   `topology = fly`
    *   A classic multi-stage interconnection network.

### Custom / Arbitrary Topologies

*   **Arbitrary Networks ([anynet.cpp](../../booksim/src/networks/anynet.cpp#L61))**:
    *   `topology = anynet`
    *   Builds custom, arbitrary graphs by reading a text-based definition file.
    *   `network_file`: Path to the text topology file.

---

## 2. Topology-Routing Compatibility Matrix

Routing functions in BookSim are heavily co-dependent on the physical topology. Setting a mismatching routing function will lead to packet loss, deadlock, or simulation runtime failures. 

> [!WARNING]
> Ensure your `routing_function` matches your `topology` configuration:

| Topology | Supported `topology` config | Valid `routing_function` parameters |
| :--- | :--- | :--- |
| **Mesh / Torus** | `mesh`, `torus` | `dim_order`, `xy_yx`, `adaptive_xy_yx`, `west_first`, `north_last`, `negative_first`, `romm`, `min_adaptive` |
| **Concentrated Mesh** | `cmesh` | `dim_order_cmesh`, `dor_cmesh` |
| **Flattened Butterfly** | `flatfly` | `ran_min_flatfly`, `adaptive_xyyx_flatfly`, `xyyx_flatfly`, `valiant_flatfly`, `ugal_flatfly`, `ugal_pni_flatfly`, `ugal_xyyx_flatfly` |
| **Dragonfly** | `dragonflynew` | `min_dragonflynew`, `ugal_dragonflynew` |
| **Trees (FatTree / QTree)** | `fattree`, `qtree`, `tree4` | `nca`, `anca` |

---

## 3. Spatial & Latency Modeling

By default, BookSim models link latency based on the layout's physical coordinates to estimate wire delay realistically.

### The `use_noc_latency` Parameter
When `use_noc_latency = 1` is configured, BookSim overrides simple single-cycle hop times by calculating spatial latencies:
- **Concentration Links**: Latency of node-to-router injection/ejection channels is calculated dynamically based on client coordinates relative to the router (e.g., in concentrated meshes and flattened butterflies).
- **Inter-Router Links**: Channel latencies are scaled proportionally to the Manhattan distance between the source and destination switch coordinates in the virtual 2D grid.

### Hidden/Hardcoded Latencies
Be aware that certain topologies have hardcoded physical latency values embedded directly in their source implementations:
- **Dragonfly (`dragonflynew`)**: In [dragonfly.cpp](../../booksim/src/networks/dragonfly.cpp#L283-L300), inter-switch channels have specific latencies built-in (active when the `DRAGON_LATENCY` macro is defined):
  - **Intra-group** local links: **10 cycles** latency.
  - **Inter-group** optical links: **100 cycles** latency.

---

## 4. Custom Networks with `anynet`

`anynet` enables you to define any arbitrary graph (e.g., rings, stars, random topologies) using a configuration text file loaded via the `network_file` parameter.

### Syntax
Each line in the topology file represents a directed link.

- **Router-to-Router link**:
  ```text
  router <src_id> port <src_port> -> router <dest_id> port <dest_port>
  ```
- **Node-to-Router injection/ejection link**:
  ```text
  node <node_id> -> router <router_id> port <router_port>
  ```

### Example: A 3-Router Ring (with 1 Concentrated Node each)
Create a text file (e.g., `triangle.txt`) with the following definition:

```text
# Inter-router links forming a ring
router 0 port 1 -> router 1 port 1
router 1 port 2 -> router 2 port 1
router 2 port 2 -> router 0 port 2

# Symmetrical reverse directions
router 1 port 1 -> router 0 port 1
router 2 port 1 -> router 1 port 2
router 0 port 2 -> router 2 port 2

# Processor nodes attached to each router
node 0 -> router 0 port 0
node 1 -> router 1 port 0
node 2 -> router 2 port 0
```

---

## 5. Diagnostic Topology Dumps

If you are writing a custom topology parser or debugging complex configurations like Dragonfly, BookSim provides built-in methods to export the compiled topology directly:

*   **`DumpChannelMap`**: Prints the full interconnection matrix of channels (`source_router,source_port,dest_router,dest_port`).
*   **`DumpNodeMap`**: Prints terminal-to-router mapping.

These functions are defined in [network.cpp](../../booksim/src/networks/network.cpp#L260-L290).

> [!TIP]
> **Programmatic Visualization**: You can redirect these output maps into a CSV file, then use a simple Python script with `networkx` and `matplotlib` to verify or graph your physical layout:
> ```python
> import networkx as nx
> import matplotlib.pyplot as plt
> import pandas as pd
> 
> # Parse the channel map CSV output by BookSim
> df = pd.read_csv("channel_map.csv", comment='#')
> G = nx.from_pandas_edgelist(df, source="source_router", target="dest_router", create_using=nx.DiGraph())
> nx.draw(G, with_labels=True)
> plt.show()
> ```

---

## 6. Fault Injection & Link Failures

BookSim provides standard support for evaluating network resilience under hardware failures:

- **`link_failures`**: An integer configuration parameter that specifies the number of random link faults to introduce in the network at simulation startup.
- **Mechanism**: The topology builder calls `InsertRandomFaults(config)` in [network.cpp](../../booksim/src/networks/network.cpp#L121). 
- *Note*: Fault injection is implemented selectively by specific topologies (e.g. standard `kncube` mesh/torus) that support fault-tolerant routing validation.