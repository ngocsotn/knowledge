# Dynamic Programming, Greedy, and Backtracking

Comprehensive DSA study guide for solving complex algorithmic problems using optimal substructures, state-space exploration, and memoization techniques.

---

## 1. Dynamic Programming (DP)

**Dynamic Programming** is an algorithmic paradigm that solves complex problems by breaking them down into simpler, overlapping subproblems, solving each subproblem exactly once, and storing their results to avoid redundant calculations.

### Core Properties of DP Problems:
1. **Overlapping Subproblems**: The same subproblems are solved repeatedly during recursion (e.g., in Fibonacci: $F(5) = F(4) + F(3)$, and $F(4) = F(3) + F(2)$ — $F(3)$ is computed twice).
2. **Optimal Substructure**: The optimal solution to the global problem can be constructed from optimal solutions to its subproblems.

### Two Approaches to DP

| Strategy | Type | Name | Mechanics |
| :--- | :--- | :--- | :--- |
| **Memoization** | **Top-Down** | Recursive | Solves the problem by calling itself recursively. If a subproblem result exists in a cache (hash map or array), returns the cached value directly. |
| **Tabulation** | **Bottom-Up** | Iterative | Fills an array or table from the smallest base cases up to the target solution. Usually faster and avoids recursion stack overflow. |

---

## 2. Greedy Algorithms

A **Greedy Algorithm** builds up a solution piece-by-piece, always choosing the next piece that offers the most obvious and immediate benefit (local optimum) in hopes of finding the global optimum.

### Core Properties of Greedy Problems:
1. **Greedy Choice Property**: A global optimum can be arrived at by selecting a local optimum at each step without ever needing to backtrack or change past choices.
2. **Optimal Substructure**: An optimal solution to the global problem contains optimal solutions to subproblems.

### DP vs. Greedy
- **DP**: Looks at all subproblems, combines their results, and decides. Highly reliable, but has $O(N)$ or $O(N^2)$ space/time complexity.
- **Greedy**: Chooses the immediate best option without considering future consequences. Extremely fast ($O(N \log N)$ or $O(N)$), but only works for specific mathematical properties.
- *Example (Coin Change)*:
  - If coin denominations are `[1, 2, 5, 10]`, a Greedy algorithm works perfectly to make change for `18` (10, 5, 2, 1).
  - If denominations are `[1, 3, 4]`, Greedy fails for `6`: Greedy chooses `4 + 1 + 1` (3 coins), whereas the optimal DP solution is `3 + 3` (2 coins).

---

## 3. Backtracking

**Backtracking** is a systematic method for searching the entire state-space of a problem to find all possible configurations or solutions. It uses **depth-first search (DFS)** to traverse a tree of states.

### Key Concept: Pruning
As the recursion traverses down a path, it checks if the current partial solution is invalid. If invalid, the algorithm **prunes** that entire subtree of choices, backtracks up the call stack, resets the state, and tries the next branch of choices. This saves massive execution time compared to brute-force searches.

```
                  Root (Start)
                 /     |     \
               A*      B      C  <-- State choices
              / \
            A1   A2(Invalid) <-- Prune this entire branch!
```

---

## 4. Coding Questions & Solutions

### Q1: The 0/1 Knapsack Problem (Classical DP)
- **Problem**: Given weights `wt` and values `val` of $N$ items, and a knapsack of capacity $W$, find the maximum value you can carry. You cannot split items (0/1).
- **Go Tabulation Solution**:
  ```go
  func knapsack(W int, wt []int, val []int, n int) int {
      // Create a 2D DP table
      dp := make([][]int, n+1)
      for i := range dp {
          dp[i] = make([]int, W+1)
      }

      // Build table dp[i][w] bottom-up
      for i := 1; i <= n; i++ {
          for w := 1; w <= W; w++ {
              if wt[i-1] <= w {
                  // Max of (include item, exclude item)
                  dp[i][w] = max(val[i-1] + dp[i-1][w-wt[i-1]], dp[i-1][w])
              } else {
                  // Exclude item
                  dp[i][w] = dp[i-1][w]
              }
          }
      }
      return dp[n][W]
  }

  func max(a, b int) int {
      if a > b { return a }
      return b
  }
  ```
  - **Time Complexity**: $O(N \times W)$
  - **Space Complexity**: $O(N \times W)$ (can be optimized to $O(W)$ using 1D array).

### Q2: Interval Scheduling (Greedy)
- **Problem**: Given a list of intervals with start and end times, find the maximum number of non-overlapping intervals you can schedule.
- **Optimal Greedy Strategy**: Always pick the interval that **ends earliest**. This leaves the maximum amount of room for subsequent intervals.
- **Go Solution**:
  ```go
  type Interval struct {
      Start, End int
  }

  func maxIntervals(intervals []Interval) int {
      if len(intervals) == 0 {
          return 0
      }

      // 1. Sort intervals by their END times ascending
      sort.Slice(intervals, func(i, j int) bool {
          return intervals[i].End < intervals[j].End
      })

      count := 1
      lastEnd := intervals[0].End

      // 2. Iterate and select non-overlapping intervals
      for i := 1; i < len(intervals); i++ {
          if intervals[i].Start >= lastEnd {
              count++
              lastEnd = intervals[i].End
          }
      }
      return count
  }
  ```
  - **Time Complexity**: $O(N \log N)$ (dominated by sorting).
  - **Space Complexity**: $O(1)$.

### Q3: Backtracking - Permutations of an Array
- **Problem**: Given an array of unique integers, return all possible permutations.
- **Go Backtracking Solution**:
  ```go
  func permute(nums []int) [][]int {
      var result [][]int
      backtrack(nums, 0, &result)
      return result
  }

  func backtrack(nums []int, start int, result *[][]int) {
      if start == len(nums) {
          // Found a complete permutation, copy and store it
          temp := make([]int, len(nums))
          copy(temp, nums)
          *result = append(*result, temp)
          return
      }

      for i := start; i < len(nums); i++ {
          // 1. Swap current index with start to set choice
          nums[start], nums[i] = nums[i], nums[start]

          // 2. Recurse down the state-space tree
          backtrack(nums, start+1, result)

          // 3. Swap back (Backtrack & reset state)
          nums[start], nums[i] = nums[i], nums[start]
      }
  }
  ```
  - **Time Complexity**: $O(N \times N!)$
  - **Space Complexity**: $O(N)$ (recursion stack depth).

---

## 5. Staff-Level Deep Dive: DP Memoization Lattices

When solving multi-dimensional DP problems (like edit-distance or path-finding on grids), the states form a mathematical structure called a **directed acyclic state-space graph**, or a **Memoization Lattice**. 

Understanding the lattice layout tells you:
1. **The State Dependencies:** Which cell must be calculated before the current cell can be evaluated.
2. **Space Compression Opportunities:** If a cell at row `i` only depends on cells in row `i-1`, the space complexity can be compressed from a 2D $O(R \times C)$ grid down to a 1D $O(C)$ sliding row.

### The Grid Pathfinding Lattice (Unique Paths II)
Consider a robot navigating an $M \times N$ grid from top-left to bottom-right, dodging obstacles.

```
State space lattice representation:
 (0,0) ──► (0,1) ──► (0,2)
   │         │         │
   ▼         ▼         ▼
 (1,0) ──► Obstacle  (1,2)
   │                   │
   ▼                   ▼
 (2,0) ──► (2,1) ──► (2,2) [Destination]
```

At any cell `(i, j)`, the optimal path count is the sum of paths from its incoming neighbors:

$$\text{dp}[i][j] = \text{dp}[i-1][j] + \text{dp}[i][j-1]$$

If an obstacle is placed at `(i, j)`, we block the lattice node, forcing `dp[i][j] = 0`.

### Go Implementation with 1D Space Compression
Instead of allocating a full $M \times N$ matrix, we can compress the space to $O(N)$ by keeping track of only the current and previous rows (sliding window).

```go
package main

import "fmt"

func uniquePathsWithObstacles(obstacleGrid [][]int) int {
	if len(obstacleGrid) == 0 || obstacleGrid[0][0] == 1 {
		return 0
	}

	cols := len(obstacleGrid[0])
	
	// Create compressed 1D lattice row
	dp := make([]int, cols)
	dp[0] = 1 // Base case: starting point

	for i := 0; i < len(obstacleGrid); i++ {
		for j := 0; j < cols; j++ {
			if obstacleGrid[i][j] == 1 {
				dp[j] = 0 // Obstacle blocks path propagation
			} else if j > 0 {
				// State-transition formula:
				// New dp[j] (representing current row's path count)
				// is sum of old dp[j] (top cell) and new dp[j-1] (left cell)
				dp[j] += dp[j-1]
			}
		}
	}
	return dp[cols-1]
}

func main() {
	grid := [][]int{
		{0, 0, 0},
		{0, 1, 0},
		{0, 0, 0},
	}

	fmt.Println("Unique paths avoiding obstacle:", uniquePathsWithObstacles(grid)) // Output: 2
}
```

---

## 6. Staff-Level Deep Dive: Bitmask Dynamic Programming (Held-Karp TSP)

When solving NP-Hard problems that involve finding an optimal subset or sequence of choices (such as the **Hamiltonian Cycle** or **Travelling Salesperson Problem**), standard recursion takes $O(N!)$ brute-force time, which is completely unscalable for $N > 12$. 

By using **Bitmask Dynamic Programming**, we can optimize this to **$O(2^N \cdot N^2)$** time by representing the visited state of nodes as a binary integer (bitmask).

### A. The Core Concept: What is a Bitmask?
Instead of passing a slow hash set or array to track visited cities (e.g., `visited = [true, false, true]`), we use a single integer's binary representation.
- A 32-bit integer can represent up to 32 boolean flags.
- **Set the $i$-th flag**: `mask | (1 << i)`
- **Check the $i$-th flag**: `(mask & (1 << i)) != 0`
- **Clear the $i$-th flag**: `mask & ^(1 << i)`

### B. TSP State-Transition Formula (Held-Karp)
Let `dp[mask][u]` represent the minimum cost of visiting the subset of cities represented by `mask`, ending at city `u`.
- To calculate `dp[mask][u]`, we find the best previous city `v` we could have traveled from:

$$\text{dp}[\text{mask}][u] = \min_{v} (\text{dp}[\text{mask} \setminus \{u\}][v] + \text{dist}[v][u])$$

where `v` is a city in `mask` other than `u`.

### C. Go Implementation of TSP (Bitmask DP)
```go
package main

import (
	"fmt"
	"math"
)

// TSP calculates the minimum cost to visit all cities and return to start
func TSP(dist [][]int) int {
	n := len(dist)
	numStates := 1 << n // 2^N states
	
	// Initialize 2D DP table: dp[mask][u]
	dp := make([][]int, numStates)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] {
			dp[i][j] = -1 // -1 means uncalculated state
		}
	}

	// Helper function for top-down memoization
	var solve func(mask, u int) int
	solve = func(mask, u int) int {
		// Base Case: If all cities are visited, return cost to go back to starting city (0)
		if mask == (1<<n)-1 {
			return dist[u][0]
		}

		if dp[mask][u] != -1 {
			return dp[mask][u]
		}

		minCost := math.MaxInt32

		// Explore next unvisited cities
		for v := 0; v < n; v++ {
			// If city v is not visited yet (v-th bit is 0)
			if (mask & (1 << v)) == 0 {
				newCost := dist[u][v] + solve(mask|(1<<v), v)
				if newCost < minCost {
					minCost = newCost
				}
			}
		}

		dp[mask][u] = minCost
		return minCost
	}

	// Start at city 0, with only city 0 visited (mask = 1)
	return solve(1, 0)
}

func main() {
	// Cost matrix between 4 cities (0-indexed)
	dist := [][]int{
		{0, 10, 15, 20},
		{10, 0, 35, 25},
		{15, 35, 0, 30},
		{20, 25, 30, 0},
	}

	fmt.Println("Shortest Travelling Salesperson Route Cost:", TSP(dist)) // Output: 80 (0->1->3->2->0: 10 + 25 + 30 + 15 = 80)
}
```

### D. Complexity Comparison: Brute Force vs. Bitmask DP
- **For $N = 20$**:
  - **Brute Force $O(N!)$**: $20! \approx 2.4 \times 10^{18}$ operations (takes **77 years** on a standard PC).
  - **Bitmask DP $O(2^N \cdot N^2)$**: $2^{20} \times 20^2 \approx 4.1 \times 10^8$ operations (takes **0.2 seconds**).
- **The Trade-off**: Bitmask DP achieves speed by trading space. The memory overhead is $O(2^N \cdot N)$, which limits this algorithm to $N \le 23$ before exhausting standard RAM.

---

## 7. Popular Interview Questions & High-Impact Answers

### Q4: What is "Bitmask DP" and when should it be utilized?
* **Answer**: **Bitmask DP** is an optimization technique that uses binary integers (bitmasks) to represent subsets of chosen elements inside dynamic programming state transitions. It should be utilized when solving NP-Hard permutation or scheduling problems (like the Travelling Salesperson Problem, Hamiltonian Paths, or subset matching) on relatively small input sizes ($N \le 22$). By mapping subset choices to bit-flags, we can compress set storage, bypass array comparisons, and reduce computational complexity from factorial $O(N!)$ to exponential $O(2^N \cdot N^2)$.

