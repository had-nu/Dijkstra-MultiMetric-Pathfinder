# Multi-Cost Pathfinding on Non-Uniform Terrain
**Dijkstra vs A(*) under a Shared Cost Model**

This repository implements and compares two canonical path planning strategies — Dijkstra and A* — applied to navigation over non-uniform terrain.

Rather than presenting the algorithms in isolation, both are evaluated under the same terrain representation and cost model, allowing a direct and technically meaningful comparison between uninformed and informed search strategies.

The project focuses on correctness, clarity of modeling, and explicit trade-offs.

---

## Problem Context

Path planning in real environments rarely optimizes geometric distance alone. Terrain slope, elevation changes, and traversal effort play a decisive role in robotics, autonomous navigation, and spatial optimization.

In this project, the environment is modeled as a discretized height map, where:

- each cell represents a spatial position;
- adjacent cells define possible movements;
- traversal cost depends on distance and elevation change.

The objective is to compute a minimum-cost path between two points, where cost approximates traversal effort rather than pure distance.

## Terrain and Cost Model

The terrain is represented as a 2D matrix of heights. From this representation, a weighted graph is constructed:

- nodes correspond to terrain cells;
- edges connect adjacent cells;
- edge weights encode a physically motivated cost function based on:
- planar distance;
- elevation difference (Δh);
- slope-related penalties.

All algorithms in this repository operate on the same graph and cost model. Differences in behavior arise solely from the search strategy.

## Dijkstra: Uninformed Optimal Search

Dijkstra’s algorithm computes the minimum-cost path in graphs with non-negative edge weights by exhaustively expanding nodes in order of accumulated cost.

Characteristics:

- guarantees optimality;
- explores the state space uniformly;
- serves as a reliable baseline for comparison;
- independent of domain-specific heuristics.

In this project, Dijkstra establishes the reference solution against which heuristic methods are evaluated.

## A*: Informed Search with Heuristic Guidance

A* extends Dijkstra by incorporating a heuristic estimate of the remaining cost to the goal.

In this implementation:

- the heuristic is admissible and consistent;
- it is derived from planar distance, optionally constrained by minimal slope assumptions;
- optimality is preserved while reducing unnecessary exploration.

A* introduces domain knowledge into the search process, trading exhaustive exploration for guided efficiency.

## Comparative Analysis

Both algorithms are evaluated under identical conditions. The comparison focuses on:

- total path cost (optimality);
- number of expanded nodes;
- relative execution time;
- sensitivity to terrain roughness;
- scenarios where heuristic guidance provides limited benefit.

The goal is not to demonstrate that one algorithm is universally superior, but to clarify when heuristic information meaningfully improves performance and when it does not.

## Repository Structure
```
Pathfinding-on-NonUniform-Terrain/
├── README.md
├── docs/
│   ├── problem_definition.md
│   ├── terrain_and_cost_model.md
│   ├── dijkstra.md
│   ├── astar.md
│   └── comparison.md
├── data/
│   ├── terrain_demo_01.csv
│   └── terrain_demo_02.csv
├── src/
│   ├── graph_model.py
│   ├── cost_model.py
│   ├── dijkstra.py
│   ├── astar.py
│   └── pathfinder.py
├── notebooks/
│   └── terrain_path_exploration.ipynb
└── tests/
    └── test_pathfinding_basic.py
```

## Usage Overview

Typical workflow:

1. Load a terrain matrix from CSV.
2. Construct the weighted graph representation.
3. Select start and goal positions.
4. Run Dijkstra or A* using the same interface.
5. Inspect the resulting path, cost, and exploration metrics.

The notebook provides visual inspection and exploratory analysis.

## Design Principles

- single problem, single cost model;
- explicit separation between algorithm and domain modeling;
- minimal abstraction layers;
- reproducible experiments;
- transparent limitations.

Performance optimization and large-scale benchmarking are deliberately out of scope.

---

## 9. References

- *Artificial Intelligence: A Modern Approach* — Russell & Norvig  
- *Introduction to Algorithms* — Cormen, Leiserson, Rivest e Stein (CLRS | Teoria dos Grafos e Algoritmos)  
- *The Art of Computer Programming, Volume 1: Fundamental Algorithms* — Donald Knuth (Busca e Estruturas de Dados)
- *Bayesian Generalized Kernel Inference for Terrain Traversability Mapping* — Shan, T., Wang, J., Englot, B., & Doherty, K. (Rough Terrain Navigation)

---

## 10. License

Educational and experimental use.

