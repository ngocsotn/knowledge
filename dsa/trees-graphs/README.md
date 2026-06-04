# Trees & Graphs

Comprehensive interview study guide covering tree and graph structures, traversals (DFS vs. BFS), Binary Search Trees, and graph representation.

---

## 1. Core Structures

### 1. Trees
A **Tree** is a hierarchical, acyclic data structure consisting of nodes connected by directed edges.
* **Binary Tree:** Every node has at most two children (`left` and `right`).
* **Binary Search Tree (BST):** A binary tree where for every node:
  * All values in its left subtree are **smaller** than the node's value.
  * All values in its right subtree are **greater** than the node's value.
  * This property allows searching, insertion, and deletion in $O(\log n)$ average time.

### 2. Graphs
A **Graph** is a non-linear data structure consisting of vertices (V) and edges (E) that connect them. Unlike trees, graphs can contain cycles and multiple disconnected pathways.
* **Representations:**
  * **Adjacency List:** An array of lists where `list[i]` stores all neighbors of vertex `i`. Space efficient: $O(V + E)$.
  * **Adjacency Matrix:** A 2D matrix where `matrix[i][j] = 1` indicates an edge between `i` and `j`. Fast lookup ($O(1)$ checks) but memory intensive ($O(V^2)$ space).

---

## 2. Traversal Strategies

To visit nodes in trees or graphs, we use two fundamental algorithms:

### 1. Breadth-First Search (BFS)
* **Concept:** Visits nodes **level-by-level**, exploring all neighbors of the current depth before moving deeper.
* **Mechanism:** Implemented iteratively using a **Queue** (FIFO).
* **Use Case:** Finding the **shortest path** in unweighted graphs.

### 2. Depth-First Search (DFS)
* **Concept:** Explores a single branch as deeply as possible before backtracking.
* **Mechanism:** Implemented recursively or iteratively using a **Stack** (LIFO).
* **Use Case:** Topological sorting, pathfinding, detecting cycles, backtracking solutions.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between BFS and DFS, and how do their memory footprints (space complexity) compare in trees?
* **Answer:**
  * **BFS** uses a Queue to explore level-by-level. Its worst-case space complexity is proportional to the **maximum width of the tree** ($O(W)$), which occurs at the bottom leaf layer (e.g., $N/2$ nodes in a balanced tree).
  * **DFS** uses a Stack (or recursion) to explore depth-first. Its worst-case space complexity is proportional to the **maximum height of the tree** ($O(H)$), which is the recursion call stack limit.
  * **Trade-off:** In wide, shallow trees, DFS uses significantly less memory than BFS. In deep, narrow trees (or skewed trees), BFS is more memory efficient.

### Q2: Why does an "In-Order Traversal" of a Binary Search Tree (BST) always return values in sorted order?
* **Answer:** By definition, a BST stores smaller values on the left, the root in the middle, and larger values on the right. In-Order traversal explores nodes in the sequence: **Left Subtree ──► Root ──► Right Subtree**. Because it recursively visits the left child (smallest), processes the root (median), and then visits the right child (largest) at every subtree level, it naturally outputs the values in strictly ascending sorted order in $O(n)$ time.

### Q3: Explain how "Breadth-First Search" (BFS) finds the shortest path in an unweighted grid (e.g., Number of Islands or Maze search).
* **Answer:** BFS is a level-order traversal. Starting at the source node, BFS explores all nodes exactly 1 unit away, then all nodes 2 units away, and so on. Since it expands outward uniformly like a ripple in water, the first time BFS encounters the target node, it is guaranteed to be the path with the **minimum number of edges (shortest path)**. To implement this:
  1. Use a Queue to store coordinate coordinates `(x, y)` and a `visited` set to prevent infinite loops.
  2. Dequeue the current cell, and check its 4 adjacent neighbors (Up, Down, Left, Right).
  3. If a neighbor is valid and unvisited, add it to the queue and mark it visited.
  * Once the queue is empty or the target is reached, execution halts.
