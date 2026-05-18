[<- Topology Index](README.md)

# Custom Arbitrary Networks (AnyNet) In-Depth Guide

The `anynet` topology allows you to load any arbitrary, custom graph (e.g., ring, star, irregular layout) from an external text file. In BookSim, this is parsed and built in the [AnyNet](../../booksim/src/networks/anynet.cpp#L61) class inside [anynet.cpp](../../booksim/src/networks/anynet.cpp).

---

## 1. Loader Parameters & File Syntax

To use custom topologies, specify the following parameters in your configuration:
```text
topology = anynet
network_file = <path_to_topology_text_file>
routing_function = min_anynet
```

### Syntax Format Rules
The text file is parsed line-by-line. Blank lines are ignored, and whitespace is used to delimit tokens:

1.  **Connecting a Router to a Router**:
    ```text
    router <src_id> router <dest_id> <latency_cycles>
    ```
    *Example:* `router 0 router 1 15` connects Router 0 to Router 1 with a directed link having a latency of 15 cycles.
    > [!IMPORTANT]
    > Links in `anynet` are **unidirectional**. If you specify a connection from `0` to `1`, the reverse path from `1` to `0` will default to a standard latency of **1 cycle** unless explicitly specified in another line.

2.  **Connecting a Processor Node to a Router**:
    ```text
    router <router_id> node <node_id> <latency_cycles>
    ```
    *Example:* `router 0 node 2 5` connects processor node 2 to router 0. Unlike router-to-router links, node-to-router statements establish **bidirectional** connections: both the injection channel and the ejection channel are assigned the designated latency (5 cycles).

---

## 2. In-Depth Network Building (`_BuildNet`)

The [_BuildNet](../../booksim/src/networks/anynet.cpp#L133) parser loads links into two primary maps inside [readFile](../../booksim/src/networks/anynet.cpp#L324):
-   `router_list[0]`: Maps `router_id` $\to$ concentrated `node_id` $\to$ `(port, latency)`.
-   `router_list[1]`: Maps `router_id` $\to$ neighboring `router_id` $\to$ `(port, latency)`.

```text
                  AnyNet Radix & Port Allocation
         +-----------------------------------------------+
         |  Ports 0 to (Concentrated Nodes - 1):         | --> Terminal Nodes
         |  Ports (Concentrated Nodes) to (Radix - 1):   | --> Neighbor Routers
         +-----------------------------------------------+
         (Ports are assigned sequentially to preserve ordering)
```

### Step 2.1: Radix Calculation & Switch Allocation
For each router index `node`, the builder calculates its total radix by summing the number of attached terminal nodes and neighboring routers:
```cpp
int radix = niter->second.size() + riter->second.size();
_routers[node] = Router::NewRouter( config, this, router_name.str( ), 
                                    node, radix, radix );
```

### Step 2.2: Terminal Node Binding
First, the builder binds injection and ejection channels for processor nodes to ports starting at index $0$:
```cpp
for(nniter = niter->second.begin(); nniter != niter->second.end(); nniter++){
  int link = nniter->first;
  (niter->second)[link].first = outport[node]; // Assign sequential port
  outport[node]++;
  
  _inject[link]->SetLatency(nniter->second.second);
  _eject[link]->SetLatency(nniter->second.second);
  _routers[node]->AddInputChannel( _inject[link], _inject_cred[link] );
  _routers[node]->AddOutputChannel( _eject[link], _eject_cred[link] );
}
```

### Step 2.3: Inter-Router Link Binding
Second, the builder binds inter-switch channels, continuing port index assignments sequentially from where terminal node bindings left off:
```cpp
for(rriter = riter->second.begin(); rriter != riter->second.end(); rriter++){
  int other_node = rriter->first;
  int link = channel_count;
  (riter->second)[other_node].first = outport[node]; // Assign sequential port
  outport[node]++;
  
  _chan[link]->SetLatency(rriter->second.second);
  _routers[node]->AddOutputChannel( _chan[link], _chan_cred[link] );
  _routers[other_node]->AddInputChannel( _chan[link], _chan_cred[link]);
  channel_count++;
}
```

---

## 3. Shortest-Path Routing via Dijkstra's Algorithm

Because `anynet` graphs are arbitrary, BookSim cannot use simple coordinate-based routing functions. Instead, it generates a custom routing table at startup using Dijkstra's algorithm:

1.  **Table Generation**: [buildRoutingTable](../../booksim/src/networks/anynet.cpp#L243) runs [route(r_start)](../../booksim/src/networks/anynet.cpp#L255) for each router.
2.  **Shortest Paths**: The solver computes the shortest path (based on hop count, not cycle delay) from every source switch to every destination processor node using **Dijkstra's Algorithm**.
3.  **Port Mapping**: The first-hop output port along the shortest path is saved into a global lookup matrix:
    ```cpp
    routing_table[r_start][dest_node] = port;
    ```
4.  **Flit Routing**: The `min_anynet` routing function simply queries this pre-computed matrix to forward flits:
    ```cpp
    int out_port = global_routing_table[r->GetID()][f->dest];
    outputs->AddRange( out_port, vcBegin, vcEnd );
    ```
    This pre-calculated Dijkstra routing lookup ensures fast simulation speeds even for highly complex custom networks.
