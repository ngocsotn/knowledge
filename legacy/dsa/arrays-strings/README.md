# Arrays & Strings

Comprehensive interview study guide covering core array and string data structures, hash maps, sliding window, and two-pointer patterns.

---

## 1. Key Array Patterns

Arrays and strings are contiguous blocks of memory, supporting $O(1)$ index lookup but requiring $O(n)$ time for arbitrary insertions or deletions.

### 1. Two-Pointer Pattern
* **Concept:** Uses two index variables that traverse the array simultaneously (typically starting from opposite ends or moving at different speeds).
* **Use Case:** Reversing arrays, checking palindromes, finding pairs in a sorted array (e.g., Two Sum II).
* **Sample Logic (Opposite ends pair sum):**
  ```javascript
  let left = 0, right = arr.length - 1;
  while (left < right) {
      let sum = arr[left] + arr[right];
      if (sum === target) return [left, right];
      else if (sum < target) left++;
      else right--;
  }
  ```

### 2. Sliding Window Pattern
* **Concept:** Maintains a dynamic window subarray boundaries (`left` and `right` pointers) that expands or shrinks based on constraints.
* **Use Case:** Subarray sum problems, longest substring without repeating characters, min/max window matches.

### 3. Prefix Sum Pattern
* **Concept:** Pre-computes a running sum array where `prefix[i] = arr[0] + ... + arr[i]`.
* **Use Case:** Answering range sum queries in $O(1)$ time (e.g., `sumRange(i, j) = prefix[j] - prefix[i-1]`).

---

## 2. Popular Interview Questions & High-Impact Answers

### Q1: How do you solve the classic "Two Sum" problem in $O(n)$ time complexity?
* **Answer:** The brute-force approach uses nested loops, taking $O(n^2)$ time. To solve it in **$O(n)$ time**, utilize a **Hash Map** to store visited values and their indices. As you loop through the array, compute the required complement (`complement = target - nums[i]`). Check if this complement exists in your hash map:
  1. If present, you have found the pair; return the complement's index and current index.
  2. If absent, insert the current number and index into the map and continue.
  * **Complexity:** $O(n)$ time (single pass, $O(1)$ map lookups) and $O(n)$ space (the hash map).

### Q2: Explain the Sliding Window algorithm to find the "Longest Substring Without Repeating Characters".
* **Answer:**
  1. Maintain a sliding window using `left` and `right` pointers, initialized to 0.
  2. Use a Hash Set (or Map) to store unique characters currently inside the active window.
  3. Expand the window by iterating `right` through the string.
  4. If the character at `right` already exists in the set, it indicates a duplicate: **shrink the window from the left** by deleting characters at `left` from the set and incrementing `left` until the duplicate character is removed.
  5. Add the character at `right` to the set, update the maximum window size (`max_len = Math.max(max_len, right - left + 1)`), and continue.
  * **Complexity:** $O(n)$ time (every character is visited at most twice) and $O(\min(n, m))$ space where $m$ is the character alphabet size.

### Q3: Why is string manipulation considered expensive in languages like Java or Python, and how do you optimize it?
* **Answer:** In Java and Python, **strings are immutable**. Every time you concatenate strings (e.g., `str = str + "a"` inside a loop), the runtime does not modify the string in-place. Instead, it allocates a completely new string object in memory and copies all existing characters over, resulting in $O(n^2)$ time complexity for a loop of size $n$. To optimize string assembly:
  * In **Java**, use **`StringBuilder`** or `StringBuffer`, which utilize mutable, dynamic buffers.
  * In **Python**, gather string pieces inside a list and combine them using **`"".join(list)`** at the end, executing in linear $O(n)$ time.
