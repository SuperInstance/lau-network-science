# lau-network-science

A comprehensive Rust library for **network science**: graph construction, generative models, centrality metrics, community detection, epidemic simulation, degree-distribution analysis, and agent-based social-network modeling.

Built on `nalgebra`, `rayon`, and `serde`. Everything is pure Rust — no C bindings, no external solvers.

---

## What This Does

`lau-network-science` gives you a toolbox for working with graphs as a scientist, not just a programmer:

- **Build graphs** — undirected and directed, with adjacency-list storage, BFS, connected components, and serialization via `serde`.
- **Generate networks** — Erdős–Rényi (G(n,p) and G(n,M)), Barabási–Albert preferential attachment, Watts–Strogatz small-world.
- **Measure centrality** — degree, betweenness (Brandes' algorithm), closeness, eigenvector (power iteration), and PageRank (undirected + directed).
- **Find communities** — Louvain modularity maximization, label propagation, modularity scoring, and Normalized Mutual Information (NMI) for comparing partitions.
- **Analyze structure** — small-world metrics (σ, γ, λ), clustering coefficient, transitivity, degree assortativity, mixing matrices, k_nn(k).
- **Simulate epidemics** — discrete SIR and SIS models on arbitrary graphs, epidemic threshold computation, averaged multi-run final-size estimation.
- **Probe resilience** — random node/edge percolation, targeted degree-based attacks, Molloy–Reed critical threshold.
- **Fit distributions** — power-law MLE (Clauset–Shalizi–Newman), CCDF, Gini coefficient, scale-free detection.
- **Model agents** — `AgentNetwork` attaches named agents with attributes to a graph, tracks weighted interactions, and computes influence rankings, bridge agents, community memberships, and full summary statistics.

---

## Key Idea

Real-world networks — social graphs, citation webs, neural connectomes, infrastructure — share universal structural fingerprints: heavy-tailed degree distributions, small-world shortcuts, community clusters, and tipping points in resilience and contagion. This library implements the core algorithms for detecting and quantifying every one of those properties, backed by peer-reviewed methods (Brandes 2001, Blondel et al. 2008, Clauset et al. 2009).

---

## Install

Add to your `Cargo.toml`:

```toml
[dependencies]
lau-network-science = "0.1"
```

Or clone directly:

```bash
git clone https://github.com/SuperInstance/lau-network-science.git
cd lau-network-science
cargo build
```

**Requirements:** Rust 2021 edition (≥ 1.56).

---

## Quick Start

### Create and analyze a small graph

```rust
use lau_network_science::{Graph, degree_centrality, clustering_coefficient};

let mut g = Graph::with_n_nodes(5);
g.add_edge(0, 1);
g.add_edge(0, 2);
g.add_edge(1, 2);
g.add_edge(2, 3);
g.add_edge(3, 4);

println!("nodes: {}, edges: {}", g.node_count(), g.edge_count());
println!("connected: {}", g.is_connected());
println!("clustering: {:.4}", clustering_coefficient(&g));

let dc = degree_centrality(&g);
println!("degree centrality: {:?}", dc);
```

### Generate a scale-free network and find communities

```rust
use lau_network_science::{barabasi_albert, louvain, betweenness_centrality};

let g = barabasi_albert(200, 3);
let communities = louvain(&g);
let unique_comms: std::collections::HashSet<usize> = communities.values().copied().collect();
println!("{} nodes split into {} communities", g.node_count(), unique_comms.len());

let bc = betweenness_centrality(&g);
let bridges: Vec<_> = bc.into_iter().filter(|(_, v)| *v > 0.1).collect();
println!("{} bridge nodes (BC > 0.1)", bridges.len());
```

### Simulate an SIR epidemic

```rust
use lau_network_science::{watts_strogatz, sir_model, epidemic_threshold};

let g = watts_strogatz(500, 6, 0.3);
let threshold = epidemic_threshold(&g);
println!("epidemic threshold: {:.4}", threshold);

let result = sir_model(&g, 0.15, 0.05, &[0], 200);
println!(
    "final size: {}/{} duration: {} steps",
    result.final_size, 500, result.duration
);
```

### Agent social network

```rust
use lau_network_science::{AgentNetwork, agent_network::Agent};
use std::collections::HashMap;

let mut net = AgentNetwork::new();
for i in 0..20 {
    net.add_agent(Agent {
        id: i,
        name: format!("Agent-{}", i),
        attributes: HashMap::new(),
    });
}
// Record interactions
net.record_interaction(0, 1);
net.record_interaction(0, 2);
net.record_interaction(1, 3);

let top = net.most_influential(5);
println!("Top influencers: {:?}", top);
let summary = net.summary();
println!("density={:.3} communities={}", summary.density, summary.num_communities);
```

---

## API Reference

### Graph Types

| Type | Description |
|------|-------------|
| `Graph` | Undirected graph (adjacency list). Supports `add_node`, `add_edge`, `remove_edge`, `remove_node`, `neighbors`, `degree`, `edges`, `bfs_distances`, `is_connected`, `connected_components`, `largest_component`. |
| `DirectedGraph` | Directed graph with separate in/out edge tracking. `add_edge(from, to)`, `successors`, `predecessors`, `out_degree`, `in_degree`, `to_undirected()`. |

Both implement `Clone, Debug, Serialize, Deserialize`.

### Generative Models (`models`)

| Function | Signature | Description |
|----------|-----------|-------------|
| `erdos_renyi` | `(n, p) → Graph` | G(n, p) — each edge exists independently with probability p |
| `erdos_renyi_gnm` | `(n, m) → Graph` | G(n, M) — exactly M uniformly random edges |
| `barabasi_albert` | `(n, m) → Graph` | Preferential attachment; starts with clique of m+1, adds n−(m+1) nodes each linking to m existing nodes |
| `watts_strogatz` | `(n, k, beta) → Graph` | Ring lattice (k neighbors), rewire each edge with probability beta |
| `directed_erdos_renyi` | `(n, p) → DirectedGraph` | Directed version of G(n, p) |

### Centrality (`centrality`)

| Function | Returns | Notes |
|----------|---------|-------|
| `degree_centrality` | `HashMap<usize, f64>` | Normalized: degree/(n−1) |
| `betweenness_centrality` | `HashMap<usize, f64>` | Brandes' O(VE) algorithm, normalized for undirected |
| `closeness_centrality` | `HashMap<usize, f64>` | Reachable/(sum of distances) |
| `eigenvector_centrality` | `HashMap<usize, f64>` | Power iteration; params: max_iter, tolerance |
| `pagerank` | `HashMap<usize, f64>` | Undirected graph, damping factor, iterative |
| `pagerank_directed` | `HashMap<usize, f64>` | Directed graph version with dangling-node handling |

### Community Detection (`community`)

| Function | Returns | Description |
|----------|---------|-------------|
| `louvain` | `HashMap<usize, usize>` | Modularity-maximizing local moves (single phase) |
| `label_propagation` | `HashMap<usize, usize>` | Sync label voting, max_iter rounds |
| `modularity` | `f64` | Newman–Girvan modularity Q |
| `get_communities` | `Vec<Vec<usize>>` | Convert membership map → community lists |
| `nmi` | `f64` | Normalized mutual information between two partitions |

### Small-World Metrics (`small_world`)

| Function | Description |
|----------|-------------|
| `average_path_length` | Mean shortest path over largest component |
| `clustering_coefficient` | Average local clustering coefficient |
| `local_clustering_coefficient` | Per-node CC: triangles / possible triangles |
| `transitivity` | Global CC: 3 × triangles / connected triples |
| `small_world_coefficient` | σ = (C/C_rand) / (L/L_rand) |
| `small_world_metrics` | Returns (σ, γ, λ) |

### Assortativity (`assortativity`)

| Function | Description |
|----------|-------------|
| `degree_assortativity` | Pearson correlation of endpoint degrees |
| `mixing_matrix` | Fraction of edges connecting degree i to degree j |
| `average_neighbor_degree` | Mean neighbor degree per node |
| `knn` | Average nearest-neighbor degree for nodes of degree k |

### Resilience (`resilience`)

| Function | Description |
|----------|-------------|
| `node_percolation` | Random node removal → `ResilienceResult` time series |
| `edge_percolation` | Random edge removal → `ResilienceResult` |
| `targeted_attack` | Degree-sorted removal (highest first) |
| `critical_threshold` | Molloy–Reed criterion: frac removed to destroy giant component |

### Epidemic Spreading (`epidemic`)

| Function | Description |
|----------|-------------|
| `sir_model` | Discrete SIR on graph → `SirResult` (time series, final size, duration) |
| `sis_model` | Discrete SIS on graph → `SisResult` |
| `epidemic_threshold` | λ_c = ⟨k⟩ / ⟨k²⟩ |
| `sir_average_final_size` | Multi-run average with random initial infection |

### Degree Distribution (`degree_distribution`)

| Function | Description |
|----------|-------------|
| `degree_histogram` | Count of nodes per degree |
| `degree_ccdf` | Complementary CDF: P(K ≥ k) |
| `power_law_fit` | MLE (alpha, x_min) via Clauset–Shalizi–Newman |
| `is_scale_free` | True if power-law fit with reasonable alpha |
| `degree_gini` | Gini coefficient of degree inequality |

### Agent Networks (`agent_network`)

| Type | Key Methods |
|------|-------------|
| `Agent` | `id`, `name`, `attributes: HashMap<String, String>` |
| `AgentNetwork` | `add_agent`, `add_communication_link`, `record_interaction`, `most_influential`, `detect_communities`, `bridge_agents`, `density`, `summary` |
| `AgentNetworkSummary` | `num_agents`, `num_links`, `density`, `clustering_coefficient`, `average_path_length`, `assortativity`, `num_communities`, `modularity` |
| `AgentNetworkAnalyzer` | `compare` (structural diff), `from_communication_log`, `random_network` |

---

## How It Works

### Graph Storage

`Graph` stores an adjacency list as `HashMap<usize, HashSet<usize>>` — sparse, unsorted node IDs, O(1) edge lookups. Edge count is tracked incrementally so `edge_count()` is O(1). `DirectedGraph` maintains parallel `out_edges` and `in_edges` maps for O(1) successor/predecessor access.

### Generative Models

- **Erdős–Rényi G(n,p):** Iterates all n(n−1)/2 pairs, flips Bernoulli(p) for each. O(n²).
- **Barabási–Albert:** Maintains a `Vec<usize>` of repeated node indices (one per incident edge endpoint) for weighted sampling in O(1). New nodes sample m targets without replacement from this list.
- **Watts–Strogatz:** Builds a ring lattice (each node connects to k/2 neighbors on each side), then rewires each edge to a random non-neighbor with probability β.

### Centrality

- **Betweenness** uses Brandes' algorithm (2001): BFS from each source, accumulate pair dependencies in reverse BFS order. O(VE) for unweighted graphs.
- **Eigenvector** uses power iteration with L2 normalization; converges when mean absolute change < tolerance.
- **PageRank** uses the standard iterative formula: `PR(v) = (1−d)/n + d × Σ(PR(u)/out_deg(u))`. Handles dangling nodes by distributing their mass uniformly.

### Community Detection

- **Louvain** (single-phase): Each node starts in its own community. Iteratively moves nodes to the neighbor's community that gives the largest modularity gain ΔQ = k_i,in/m − Σ_tot·k_i/(2m²). Repeats until no improvement.
- **Label Propagation:** Each node adopts the most common label among its neighbors. Nodes are processed in random order. Terminates when labels stabilize or max iterations reached.

### Epidemic Models

Discrete-time stochastic simulation. At each step:
- **SIR:** Each infected node infects each susceptible neighbor with probability β, then recovers with probability γ. Recovered nodes are permanently immune.
- **SIS:** Same infection step, but recovered nodes return to susceptible (no immunity).

The **epidemic threshold** λ_c = ⟨k⟩/⟨k²⟩ comes from the mean-field theory of network epidemiology (Pastor-Satorras et al.). If β/γ > λ_c, epidemics can spread globally.

### Resilience

Percolation analysis removes nodes/edges incrementally and tracks the largest component fraction at each step. The **Molloy–Reed criterion** gives the critical threshold: f_c = 1 − 1/(⟨k²⟩/⟨k⟩ − 1). Above f_c, no giant component exists.

### Power-Law Fitting

Implements the Clauset–Shalizi–Newman method: for each candidate x_min, compute the MLE exponent α = 1 + n/Σ(ln(x_i/(x_min−0.5))), then pick the x_min minimizing the Kolmogorov–Smirnov statistic between empirical and fitted CDFs.

---

## The Math

### Modularity

$$Q = \frac{1}{2m}\sum_{ij}\left[A_{ij} - \frac{k_i k_j}{2m}\right]\delta(c_i, c_j)$$

where $m$ is the number of edges, $A_{ij}$ the adjacency matrix, $k_i$ the degree of node $i$, and $\delta(c_i,c_j)=1$ when nodes share a community.

### Betweenness Centrality

$$C_B(v) = \sum_{s \neq v \neq t} \frac{\sigma_{st}(v)}{\sigma_{st}}$$

where $\sigma_{st}$ is the total number of shortest paths from $s$ to $t$, and $\sigma_{st}(v)$ is the number passing through $v$.

### PageRank

$$PR(v) = \frac{1-d}{N} + d \sum_{u \in \text{in}(v)} \frac{PR(u)}{L(u)}$$

with damping factor $d$ (typically 0.85), $N$ nodes, and $L(u)$ the out-degree of $u$.

### Epidemic Threshold

$$\lambda_c = \frac{\langle k \rangle}{\langle k^2 \rangle}$$

An epidemic spreads if the effective transmission rate $\beta/\gamma$ exceeds $\lambda_c$.

### Molloy–Reed Criterion

A random graph with degree distribution $p_k$ has a giant component when:

$$\frac{\langle k^2 \rangle}{\langle k \rangle} > 2$$

The critical removal fraction is $f_c = 1 - 1/(\kappa - 1)$ where $\kappa = \langle k^2\rangle/\langle k\rangle$.

### Power-Law MLE

For a discrete power law $P(k) \propto k^{-\alpha}$ with $k \geq k_{\min}$:

$$\hat{\alpha} = 1 + n \left[\sum_{i=1}^{n} \ln \frac{k_i}{k_{\min} - 0.5}\right]^{-1}$$

### Small-World Coefficient

$$\sigma = \frac{C/C_{\text{rand}}}{L/L_{\text{rand}}}$$

A network is "small-world" if $\sigma \gg 1$: high clustering like a lattice, short paths like a random graph.

---

## Tests

63 integration tests cover all modules:

```bash
cargo test
```

Tests verify graph construction, model properties (e.g., BA graphs are connected, WS graphs have correct edge count), centrality convergence, community detection on known structures, epidemic dynamics, and degree-distribution fitting.

---

## License

MIT
