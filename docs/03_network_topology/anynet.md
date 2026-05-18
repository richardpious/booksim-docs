[<- Topology Index](README.md)

# Custom Arbitrary Networks (AnyNet) Topology

The `anynet` topology allows you to load any arbitrary, custom graph (e.g., ring, star, irregular layout) from an external text file. It is implemented in the [AnyNet](../../booksim/src/networks/anynet.cpp#L61) class inside [anynet.cpp](../../booksim/src/networks/anynet.cpp).

---

## 1. Configuration & Text File Syntax

To load a custom graph, specify the following parameters:
```text
topology = anynet
network_file = <path_to_topology_text_file>
routing_function = min_anynet
```

### 1.1 Parser Syntax Rules
The parser reads the text file line-by-line, ignoring blank lines and comments (`#`). There are two primary connection statements:

-   **Router-to-Router Link (Unidirectional)**:
    ```text
    router <src_id> router <dest_id> <latency_cycles>
    ```
    *Note:* Inter-switch links in `anynet` are unidirectional. The reverse path requires an explicit statement, otherwise it defaults to a latency of **1 cycle**.
-   **Router-to-Node Link (Bidirectional)**:
    ```text
    router <router_id> node <node_id> <latency_cycles>
    ```
    *Note:* Node statements are bidirectional; both the injection and ejection channels are assigned the parsed cycle latency.

---

## 2. Network Construction (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/anynet.cpp#L133) parser loads links into two primary maps inside [readFile](../../booksim/src/networks/anynet.cpp#L324):
-   `router_list[0]`: Maps `router_id` $\to$ concentrated `node_id` $\to$ `(port, latency)`.
-   `router_list[1]`: Maps `router_id` $\to$ neighboring `router_id` $\to$ `(port, latency)`.

### 2.1 Radix Sizing & Switch Allocation
For each router index `node`, switch radix is computed from the total number of connected processor nodes and neighbor switches:
```cpp
int radix = niter->second.size() + riter->second.size();
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, radix, radix );
```

### 2.2 Terminal Node Port Bindings
Processor node injection/ejection channels are bound to switch ports sequentially starting from $0$:
```cpp
for(nniter = niter->second.begin(); nniter != niter->second.end(); nniter++){
  int link = nniter->first;
  (niter->second)[link].first = outport[node];
  outport[node]++;
  
  _inject[link]->SetLatency(nniter->second.second);
  _eject[link]->SetLatency(nniter->second.second);
  _routers[node]->AddInputChannel( _inject[link], _inject_cred[link] );
  _routers[node]->AddOutputChannel( _eject[link], _eject_cred[link] );
}
```

### 2.3 Inter-Router Link Port Bindings
Inter-switch channels are bound continuing port index assignments sequentially where terminal node bindings left off:
```cpp
for(rriter = riter->second.begin(); rriter != riter->second.end(); rriter++){
  int other_node = rriter->first;
  int link = channel_count;
  (riter->second)[other_node].first = outport[node];
  outport[node]++;
  
  _chan[link]->SetLatency(rriter->second.second);
  _routers[node]->AddOutputChannel( _chan[link], _chan_cred[link] );
  _routers[other_node]->AddInputChannel( _chan[link], _chan_cred[link]);
  channel_count++;
}
```

---

## 3. Pre-Computed Routing Table Setup

Because custom networks are arbitrary, coordinate-based routing functions cannot be used. Instead, BookSim pre-computes shortest paths at startup:

1.  **Dijkstra's Solver**: [buildRoutingTable](../../booksim/src/networks/anynet.cpp#L243) runs [route(r_start)](../../booksim/src/networks/anynet.cpp#L255) for each switch.
2.  **Shortest Paths**: The solver computes the shortest path (based on hop count, not link delay) from all switches to all destination terminal nodes using **Dijkstra's Algorithm**.
3.  **Port Lookup Table**: The first-hop output port along the calculated shortest path is saved to the routing table:
    ```cpp
    routing_table[r_start][dest_node] = port;
    ```
4.  **Routing Function**: The `min_anynet` routing function queries this pre-computed matrix to forward flits:
    ```cpp
    int out_port = global_routing_table[r->GetID()][f->dest];
    outputs->AddRange( out_port, vcBegin, vcEnd );
    ```
