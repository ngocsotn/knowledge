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

---

## 4. Advanced Shortest Path & Query Optimization

For weighted graphs and multi-dimensional coordinate fields, standard BFS/DFS algorithms are too slow or unable to resolve optimal paths. We rely on advanced graph search and tree query architectures.

### 1. Dijkstra's Shortest Path (Weighted Graphs)
* **Goal:** Finds the absolute shortest path from a source node to all other nodes in a graph with non-negative edge weights.
* **Mechanism:** Relies on a **Min-Priority Queue** (usually a Binary Min-Heap) to greedily pull the next closest unvisited node.
* **Complexity:** $O((V + E) \log V)$ with adjacency list and binary heap.

#### Go Implementation of Dijkstra
```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

// Edge represents a weighted connection
type Edge struct {
	To, Weight int
}

// Item in our Priority Queue
type Item struct {
	Node, Dist int
	index      int
}

type PriorityQueue []*Item

func (pq PriorityQueue) Len() int           { return len(pq) }
func (pq PriorityQueue) Less(i, j int) bool { return pq[i].Dist < pq[j].Dist }
func (pq PriorityQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}
func (pq *PriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*Item)
	item.index = n
	*pq = append(*pq, item)
}
func (pq *PriorityQueue) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	old[n-1] = nil
	item.index = -1
	*pq = old[0 : n-1]
	return item
}

// Dijkstra calculates shortest distances from source to all nodes
func Dijkstra(graph [][]Edge, src int) []int {
	n := len(graph)
	dist := make([]int, n)
	for i := range dist {
		dist[i] = math.MaxInt32
	}
	dist[src] = 0

	pq := make(PriorityQueue, 0)
	heap.Init(&pq)
	heap.Push(&pq, &Item{Node: src, Dist: 0})

	visited := make([]bool, n)

	for pq.Len() > 0 {
		curr := heap.Pop(&pq).(*Item)
		u := curr.Node

		if visited[u] {
			continue
		}
		visited[u] = true

		for _, edge := range graph[u] {
			v := edge.To
			weight := edge.Weight

			if !visited[v] && dist[u]+weight < dist[v] {
				dist[v] = dist[u] + weight
				heap.Push(&pq, &Item{Node: v, Dist: dist[v]})
			}
		}
	}
	return dist
}

func main() {
	// Simple graph representation
	n := 5
	graph := make([][]Edge, n)
	graph[0] = []Edge{{To: 1, Weight: 4}, {To: 2, Weight: 1}}
	graph[1] = []Edge{{To: 3, Weight: 1}}
	graph[2] = []Edge{{To: 1, Weight: 2}, {To: 3, Weight: 5}}
	graph[3] = []Edge{{To: 4, Weight: 3}}
	
	distances := Dijkstra(graph, 0)
	fmt.Println("Shortest path distances from Node 0:", distances) // Output: [0, 3, 1, 4, 7]
}
```

---

### 2. A* Pathfinding (Heuristic-Guided Search)
Dijkstra searches in all directions uniformly, which wastes CPU cycles in massive grids. **A* (A-Star)** optimizes this by directing search towards the target using an estimating **Heuristic Function** $h(n)$ (like Manhattan or Euclidean distance):

$$f(n) = g(n) + h(n)$$

* **$g(n)$:** The exact cost path from start to current node $n$.
* **$h(n)$:** The estimated cost from current node $n$ to the target.
* **$f(n)$:** The total estimated cost of path through node $n$.

#### Go Implementation of A* on 2D Coordinate Grid
```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Point struct {
	X, Y int
}

// Manhattan distance heuristic
func Heuristic(p1, p2 Point) int {
	return int(math.Abs(float64(p1.X-p2.X)) + math.Abs(float64(p1.Y-p2.Y)))
}

type Node struct {
	Pos       Point
	G, H, F   int
	Parent    *Node
	index     int
}

type NodePQ []*Node
func (pq NodePQ) Len() int           { return len(pq) }
func (pq NodePQ) Less(i, j int) bool { return pq[i].F < pq[j].F }
func (pq NodePQ) Swap(i, j int)      { pq[i], pq[j] = pq[j], pq[i]; pq[i].index = i; pq[j].index = j }
func (pq *NodePQ) Push(x interface{}) {
	n := len(*pq)
	node := x.(*Node)
	node.index = n
	*pq = append(*pq, node)
}
func (pq *NodePQ) Pop() interface{} {
	old := *pq
	n := len(old)
	node := old[n-1]
	old[n-1] = nil
	node.index = -1
	*pq = old[0 : n-1]
	return node
}

func AStar(grid [][]int, start, target Point) []Point {
	rows := len(grid)
	cols := len(grid[0])
	
	openSet := make(NodePQ, 0)
	heap.Init(&openSet)
	
	startNode := &Node{
		Pos: start,
		G:   0,
		H:   Heuristic(start, target),
	}
	startNode.F = startNode.G + startNode.H
	heap.Push(&openSet, startNode)
	
	closedSet := make(map[Point]bool)
	dirs := []Point{{-1, 0}, {1, 0}, {0, -1}, {0, 1}} // Up, Down, Left, Right

	for openSet.Len() > 0 {
		curr := heap.Pop(&openSet).(*Node)
		
		if curr.Pos == target {
			// Backtrack path
			path := []Point{}
			for curr != nil {
				path = append([]Point{curr.Pos}, path...)
				curr = curr.Parent
			}
			return path
		}
		
		closedSet[curr.Pos] = true
		
		for _, d := range dirs {
			neighborPos := Point{curr.Pos.X + d.X, curr.Pos.Y + d.Y}
			
			// Validate bounds & obstacles (e.g. grid[x][y] == 1 means obstacle)
			if neighborPos.X < 0 || neighborPos.X >= rows || neighborPos.Y < 0 || neighborPos.Y >= cols || grid[neighborPos.X][neighborPos.Y] == 1 {
				continue
			}
			if closedSet[neighborPos] {
				continue
			}
			
			gScore := curr.G + 1 // Assumes step weight is 1
			
			neighborNode := &Node{
				Pos:    neighborPos,
				G:      gScore,
				H:      Heuristic(neighborPos, target),
				Parent: curr,
			}
			neighborNode.F = neighborNode.G + neighborNode.H
			
			heap.Push(&openSet, neighborNode)
		}
	}
	return nil // No path found
}

func main() {
	// Grid representation (0 = Path, 1 = Wall)
	grid := [][]int{
		{0, 0, 0, 0, 0},
		{0, 1, 1, 1, 0},
		{0, 0, 0, 0, 0},
	}
	path := AStar(grid, Point{0, 0}, Point{2, 4})
	fmt.Println("Path resolved by A*:", path)
}
```

---

### 3. Segment Trees (Range Query Optimizations)
A **Segment Tree** is a static tree structure designed to handle range queries (e.g., finding the sum, minimum, or maximum of an array slice) and point updates in **$O(\log n)$ time**, bypassing the traditional $O(n)$ array scan bottlenecks.

```
Array: [1, 3, 5, 7, 9, 11]

                  Sum: [0-5] = 36
                 /               \
         Sum: [0-2] = 9        Sum: [3-5] = 27
         /          \          /             \
    [0-1] = 4     [2] = 5  [3-4] = 16       [5] = 11
    /       \              /        \
 [0]=1     [1]=3        [3]=7      [4]=9
```

#### Go Segment Tree Implementation (Range Sum Query)
```go
package main

import "fmt"

type SegmentTree struct {
	tree []int
	n    int
}

func NewSegmentTree(arr []int) *SegmentTree {
	n := len(arr)
	st := &SegmentTree{
		tree: make([]int, 4*n), // 4N allocation prevents out-of-bounds leaf children access
		n:    n,
	}
	st.build(arr, 0, 0, n-1)
	return st
}

func (st *SegmentTree) build(arr []int, node, start, end int) {
	if start == end {
		st.tree[node] = arr[start]
		return
	}
	mid := (start + end) / 2
	st.build(arr, 2*node+1, start, mid)
	st.build(arr, 2*node+2, mid+1, end)
	st.tree[node] = st.tree[2*node+1] + st.tree[2*node+2] // Range Sum Formula
}

// QueryRange fetches sum in [L, R] in O(log N)
func (st *SegmentTree) QueryRange(L, R int) int {
	return st.query(0, 0, st.n-1, L, R)
}

func (st *SegmentTree) query(node, start, end, L, R int) int {
	if R < start || end < L {
		return 0 // Out of range bound
	}
	if L <= start && end <= R {
		return st.tree[node] // Node fully encapsulated within query slice
	}
	mid := (start + end) / 2
	p1 := st.query(2*node+1, start, mid, L, R)
	p2 := st.query(2*node+2, mid+1, end, L, R)
	return p1 + p2
}

// UpdatePoint updates value at array index idx to val in O(log N)
func (st *SegmentTree) UpdatePoint(idx, val int) {
	st.update(0, 0, st.n-1, idx, val)
}

func (st *SegmentTree) update(node, start, end, idx, val int) {
	if start == end {
		st.tree[node] = val
		return
	}
	mid := (start + end) / 2
	if start <= idx && idx <= mid {
		st.update(2*node+1, start, mid, idx, val)
	} else {
		st.update(2*node+2, mid+1, end, idx, val)
	}
	st.tree[node] = st.tree[2*node+1] + st.tree[2*node+2]
}

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	st := NewSegmentTree(arr)
	
	fmt.Println("Sum of range [1, 3]:", st.QueryRange(1, 3)) // output: 3 + 5 + 7 = 15
	
	st.UpdatePoint(1, 10) // index 1 updated to 10
	fmt.Println("Sum of range [1, 3] after update:", st.QueryRange(1, 3)) // output: 10 + 5 + 7 = 22
}
```

