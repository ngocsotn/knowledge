# Sorting & Searching

Comprehensive interview study guide covering core sorting algorithms (Merge Sort, Quick Sort), Binary Search, and search space optimization.

---

## 1. Core Sorting Algorithms

| Algorithm | Best Time | Average Time | Worst Time | Space Complexity | Stable? | Key Feature |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Merge Sort** | $O(n \log n)$ | $O(n \log n)$ | $O(n \log n)$ | $O(n)$ auxiliary | **Yes** | Guaranteed $O(n \log n)$ time but uses extra memory. |
| **Quick Sort** | $O(n \log n)$ | $O(n \log n)$ | $O(n^2)$ | $O(\log n)$ stack | No | In-place, fast in practice. Brittle worst case. |
| **Insertion Sort**| $O(n)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ auxiliary | **Yes** | Extremely fast for small or nearly sorted arrays. |

---

## 2. Binary Search ($O(\log n)$)

**Binary Search** is an optimal searching algorithm that halves the search space at each step.
* **Prerequisite:** The input array **must be sorted**.
* **Mechanism:**
  1. Set `low = 0` and `high = arr.length - 1`.
  2. Compute `mid = low + Math.floor((high - low) / 2)`.
  3. If `arr[mid] === target`, return index.
  4. If `arr[mid] < target`, shift `low = mid + 1` (discard left half).
  5. If `arr[mid] > target`, shift `high = mid - 1` (discard right half).

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between "Merge Sort" and "Quick Sort", and how do you choose between them?
* **Answer:**
  * **Merge Sort** utilizes a **Divide-and-Conquer** strategy. It splits the array in half recursively, sorts the halves, and merges them using $O(n)$ extra space. It is **stable** (preserves the relative order of duplicate elements) and guarantees $O(n \log n)$ performance under any input layout.
  * **Quick Sort** is an **in-place** sorting algorithm. It selects a pivot, partitions elements around it, and recurses. It is **unstable** and can degrade to $O(n^2)$ if the pivot is chosen poorly.
  * **Choice:** Choose Quick Sort for general in-memory sorting of arrays where space is constrained. Choose Merge Sort when stability is critical (e.g., sorting database records with multiple keys) or when sorting Linked Lists (since list node pointers can be merged without allocating $O(n)$ extra contiguous arrays).

### Q2: How do you perform Binary Search on a "Rotated Sorted Array" (e.g., `[4,5,6,7,0,1,2]`)?
* **Answer:** You can still solve this in **$O(\log n)$ time** using a modified Binary Search:
  1. Compute `mid` as usual. If `nums[mid] === target`, return index.
  2. Determine which half of the array is **normally sorted**:
     * If `nums[low] <= nums[mid]`, the **left half is sorted**.
     * Else, the **right half is sorted**.
  3. Search inside the sorted half:
     * If the left half is sorted, check if target falls within bounds (`nums[low] <= target && target < nums[mid]`). If yes, search left (`high = mid - 1`); else search right (`low = mid + 1`).
     * If the right half is sorted, check if target falls within bounds (`nums[mid] < target && target <= nums[high]`). If yes, search right (`low = mid + 1`); else search left (`high = mid - 1`).
  4. Repeat until bounds cross.

### Q3: Why is computing middle index using `mid = (low + high) / 2` discouraged, and what is the correct syntax?
* **Answer:** In languages with fixed-size integer limits (like Java, C++, or Go, where standard integers are 32-bit or 64-bit), if `low` and `high` are extremely large values (close to the maximum integer limit), adding them together (`low + high`) can exceed the maximum range, triggering an **integer overflow** bug. This causes the value to wrap around to a negative number, leading to out-of-bounds index errors. The correct, overflow-safe syntax is:
  `mid = low + Math.floor((high - low) / 2)` (or `low + (high - low) / 2` in typed languages). This computes the offset distance first, ensuring the calculation never exceeds `high`.
