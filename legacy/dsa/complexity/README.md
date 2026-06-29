# Big O & Complexity Analysis

Comprehensive interview study guide covering Big O notation, time/space complexity, recursion, and algorithm analysis heuristics.

---

## 1. Complexity Classes

Big O notation describes the upper bound of an algorithm's execution time or memory footprint as the input size ($n$) grows toward infinity.

| Complexity | Class Name | Growth Curve Description | Standard Example |
| :--- | :--- | :--- | :--- |
| **$O(1)$** | Constant | Time stays flat regardless of input size. | Array index lookup, Hash Map read. |
| **$O(\log n)$** | Logarithmic | Time increases minimally as size doubles. | Binary Search. |
| **$O(n)$** | Linear | Time increases proportionally with input. | Single loop over an array, string scan. |
| **$O(n \log n)$** | Linearithmic | Standard optimal sorting complexity. | Merge Sort, Quick Sort (average). |
| **$O(n^2)$** | Quadratic | Nested loop growth. High overhead. | Bubble Sort, nested nested loops. |
| **$O(2^n)$** | Exponential | Doubling iterations per increment of $n$. | Recursive Fibonacci, power set subsets. |
| **$O(n!)$** | Factorial | Extreme growth. Impractical at scale. | Traveling Salesperson (brute force). |

---

## 2. Complexity Analysis Heuristics

To evaluate time complexity during interviews:
* **Drop Constants:** $O(2n + 5)$ simplifies strictly to $O(n)$.
* **Keep Dominant Terms:** $O(n^2 + n)$ simplifies strictly to $O(n^2)$ because as $n \to \infty$, the $n^2$ term dwarfs the linear $n$ term.
* **Space Complexity:** Measures the **additional auxiliary memory** allocated by your algorithm (excluding the input arguments).
  * In-place modification = $O(1)$ space.
  * Allocating a clone array or a recursion call stack = $O(n)$ space.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: How do you calculate the Time and Space complexity of a recursive function?
* **Answer:** Recursive complexity is analyzed using the call tree:
  1. **Time Complexity:** Multiply the total number of recursive calls made by the time complexity of a single call. For example, recursive Fibonacci `fib(n) = fib(n-1) + fib(n-2)` splits into 2 calls per level, leading to a tree depth of $n$ and a time complexity of $O(2^n)$.
  2. **Space Complexity:** Determined by the **maximum depth of the recursion call stack** at any given moment. Each active function call allocates a stack frame. Even if a function makes $O(2^n)$ total calls, if the maximum tree depth is $n$, the space complexity is strictly $O(n)$.

### Q2: What is the difference between "Average-Case" and "Worst-Case" complexity, and why does Quick Sort have different ratings?
* **Answer:**
  * **Worst-Case:** The absolute maximum iterations an algorithm could run under the most hostile input layout.
  * **Average-Case:** The expected behavior of the algorithm under randomized inputs.
  * **Quick Sort case:** Quick Sort relies on selecting a "pivot" element to partition the array.
    * **Average-Case is $O(n \log n)$:** When the pivot splits the array roughly in half at each step.
    * **Worst-Case is $O(n^2)$:** When the input is already sorted (or reverse sorted) and the pivot is chosen poorly (e.g., always selecting the first or last element), reducing partition sizes by only 1 element per step, resulting in $n$ recursive levels.

### Q3: What is "Amortized" time complexity, and how does it apply to dynamic arrays (like ArrayList/Vector)?
* **Answer:** **Amortized** complexity is the average time taken per operation over a long sequence of operations, where a single rare, expensive operation is "smoothed out" by a massive number of cheap operations. For example, inserting into a dynamic array (like `push` in JS arrays) is typically **$O(1)$ amortized**:
  * Most pushes just write to an available pre-allocated index in $O(1)$ time.
  * When the array capacity is full, the next push triggers a resize operation—allocating a new double-sized array and copying all $n$ elements, taking $O(n)$ time.
  * Because resizing occurs extremely rarely (only after $n$ cheap pushes), the average cost per insertion remains $O(1)$.
