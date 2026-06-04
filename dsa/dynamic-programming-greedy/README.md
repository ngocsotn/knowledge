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
