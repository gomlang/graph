# ecosystem::graph

A generic mutable multigraph library implemented in GoML. The module supports
directed and undirected graphs with arbitrary node and edge values, parallel
edges, self-loops, stable handles, graph transformations and weighted algorithms.

```gom
use ecosystem::graph;

fn route_cost() -> Result[Option[i64], graph::Error] {
    let roads: graph::Graph[string, i64] = graph::Graph::new(false);
    let a = roads.add_node("a");
    let b = roads.add_node("b");
    let _ = roads.add_edge(a, b, 7)?;
    let paths = graph::dijkstra(roads, a, |distance| distance)?;
    Result::Ok(paths.distance_to(b))
}
```

## Graph operations

- `Graph[N, E]::new(directed)`, `is_directed`, `node_count`, `edge_count`.
- `add_node`, `add_edge`, `node`, `edge`, `set_node`, `set_edge`.
- `remove_node` also removes every incident edge; `remove_edge` and `clear`.
- `contains_node`, `contains_edge`, `nodes`, `edges`, `neighbors`,
  `predecessors`, `edges_between`.
- `map[A, B](node_map, edge_map)` and `copy` create an independent graph and
  return `Copy` with old-to-new node and edge handle maps.

`NodeId` and `EdgeId` are opaque, hashable, graph-specific handles. Their `index`
is useful for ordering and diagnostics, but is not their complete identity.
Passing another graph's handle returns `WrongGraph`; removed handles return
`InvalidNode` or `InvalidEdge`. Slots are never reused, including after `clear`.
Deletion releases labels but retains slot metadata. Copying into a fresh graph
compacts slots and changes every handle's identity.

Graphs have shared mutable storage: assigning or passing a graph preserves its
identity and mutations. `copy` duplicates topology and label values; labels that
contain references still share their referenced data. `nodes`, `edges` and
adjacency methods return independent outer vectors. Undirected adjacency emits
a self-loop once and keeps each parallel edge. Directed adjacency follows
insertion order; undirected adjacency lists outgoing edges before incoming ones.

## Algorithms

| API | Behavior |
| --- | --- |
| `bfs`, `dfs` | Iterative reachable-node traversal in deterministic order |
| `unweighted_path` | Minimum-edge path, including both endpoints; `None` when unreachable |
| `topological_sort` | Directed Kahn ordering; detects cycles including self-loops |
| `weak_components` | Components ignoring directed edge orientation |
| `strong_components` | Iterative Kosaraju SCCs; ordinary components for undirected graphs |
| `dijkstra` | Nonnegative `i64` weights, heap frontier, all reachable distances |
| `astar` | Targeted path, cached nonnegative heuristic, reopening of improved nodes |
| `bellman_ford` | Signed weights and reachable negative-cycle detection |
| `minimum_spanning_forest` | Undirected Kruskal forest; negative weights supported |
| `UnionFind` | Checked indices, path compression, union by size, incremental additions |

`Paths` exposes `start`, `distance_to`, `path_to`, `route_to`, and `reachable`.
`Route` contains node handles, exact edge handles and total cost, so parallel
edges are unambiguous. Unreachable targets return `None`; a source-to-source
route has one node, no edges and zero cost. Results describe the graph at search
time; later mutation can invalidate returned handles.

Weight callbacks are evaluated once per live edge. Dijkstra and A* reject
negative weights even in disconnected components. A* evaluates the heuristic
once per live node, rejects negative values and requires zero at the target.
The caller must supply an admissible heuristic for optimality; inconsistent
admissible heuristics are supported through reopening. Bellman–Ford ignores
unreachable negative cycles. In an undirected graph, a reachable negative edge
forms a negative closed walk and is reported as a negative cycle.

All cost accumulation is checked: intermediate overflow returns `Overflow`,
including an explored candidate that would not improve the final path. No
integer value is reserved as an infinity sentinel. `Forest` reports selected
edges, total cost and component count, including isolated live nodes; empty
graphs have zero components. Equal-weight forest edges are ordered by edge slot.

`directed_cycle(graph)` returns `Some(Vec[EdgeId])` containing one directed
cycle in traversal order, or `None` when the graph is acyclic. Consecutive edges
connect and the last edge closes back to the first source. A self-loop is a
one-edge cycle; parallel edges keep their distinct IDs. The search visits live
nodes in slot order and follows stored adjacency order, so repeated calls on an
unchanged graph return the same witness. The graph must be directed; undirected
inputs return `RequiresDirected`. Search uses an explicit DFS stack and takes
O(S + V + E) time and O(V + E) auxiliary storage without recursing on graph depth.
Returned handles describe the graph at search time and may be invalidated by
later mutation.

## Complexity and constraints

Let S be allocated node/edge slots (including deleted slots), V live nodes, and
E live edges. Enumeration scans the relevant slot array. Traversals and SCCs
take O(S + V + E); Dijkstra takes O(S + (V + E) log(V + E)) with a lazy heap;
Bellman–Ford takes O(S + VE); Kruskal takes O(S + E log E). A* can revisit nodes
with inconsistent heuristics. Adjacency lookup copies O(degree) values. Edge
deletion scans its endpoint adjacency vectors. Node deletion marks all incident
edges once and filters each affected neighbor's adjacency once, taking
O(degree + total adjacency size of affected neighbors), including parallel
edges and loops. `clear` takes O(S + E), invalidates every old handle, and retains
slot history so later insertions never revive deleted handles. Both operations
preserve graph aliases and the adjacency order of surviving edges.

`undirected_cycle(graph)` provides the equivalent witness for undirected graphs.
It returns `None` for a forest, one edge for a self-loop, or two distinct edges
for parallel connections. Witness edges are ordered around the cycle, but an
edge's stored `from`/`to` direction can oppose traversal. The search skips only
the exact parent edge and preserves edge identity, including after deletions.
Directed input returns `RequiresUndirected`. The deterministic depth-first search
uses an explicit stack, O(V + E) work and O(V + E) auxiliary storage over live
nodes and adjacency (enumerating graph nodes also visits retained deleted slots).
An exhaustive four-node graph test checks cycle existence against independent
transitive-closure/component counts, alongside multiedge, self-loop and deep-graph
regressions.

Callbacks must not mutate the graph being processed. Shared storage is not
synchronized for concurrent access. Costs are `i64`; floating weights, graph
serialization, flow algorithms and dynamic shortest-path maintenance are outside
this API. Traversal and SCC implementations do not recurse on graph depth.

## Validation

The public API suite checks mutations, cross-graph identity, stale handles,
parallel edges, loops, generic maps, paths, overflow, negative cycles and forest
behavior. Independent checks include all 64 loop-free directed three-node
graphs, 80 deterministic weighted graphs compared against Floyd–Warshall from
every source, 32 complete four-node graphs compared against exhaustive spanning
tree subsets, A* reopening and a 3,000-node chain/cycle.

`examples/basic` supplies its own label types. Run
`(cd ../verification && just ecosystem-test graph)` from this library repository to
check the library and example, verify an independent downstream snapshot,
build the example twice, verify cached artifact stability, and run the executable.

## Development and examples

Requires GoML 0.1.56 or newer. The `examples/basic/` example shares the root manifest. From the library root, run:

```sh
goml run --example basic
goml test
goml verify --timeout 300s
```

`goml test` builds the example and runs its tests. `goml verify` repeats the example checks as an independent module against an isolated registry snapshot. `(cd ../verification && just ecosystem-test graph)` also retains the library-specific smoke and compatibility checks.

### Dependency generations

`topological_generations(graph)` returns `Vec[Vec[NodeId]]`: all zero-indegree
nodes are generation zero, and every remaining node is in the earliest generation
after all its predecessors. Equivalently, the generation index is the longest
path length from any source. Nodes in one generation have no dependencies on each
other and can be scheduled in parallel once earlier generations have completed.
Each generation is sorted by ascending stable node index, independent of edge
insertion order; isolated nodes belong to generation zero.

The graph must be directed. Parallel edges each contribute to indegree, removed
nodes/edges are skipped, and a cycle anywhere returns `ErrorKind::Cycle` without a
partial result. An empty directed graph returns no generations. The graph is not
modified. This iterative algorithm uses O(S + E + V log V) time and O(V + E) additional
storage (including returned nodes and temporary adjacency lists), where S counts
retained node slots, V live nodes and E live edges. It has no recursive call depth. The existing
`topological_sort` retains its FIFO ordering.
