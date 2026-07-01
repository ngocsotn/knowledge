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
  `mid = low + (high - low) / 2`. This computes the offset distance first, ensuring the calculation never exceeds `high`.

---

## 4. Number Theory & Cryptographic Math: Euclidean Algorithms

When designing secure APIs, parsing cryptographic keys (like RSA), or resolving grid cycles, we rely on fundamental number theory search algorithms.

### A. The Basic Euclidean Algorithm (GCD)
- **Goal**: Computes the **Greatest Common Divisor (GCD)** of two integers $a$ and $b$ (the largest integer that divides both without remainder).
- **Mathematical Invariant**: 
  $$\gcd(a, b) = \gcd(b, a \bmod b)$$
- **Complexity**: $O(\log \min(a, b))$ (the number of steps is logarithmic in the size of the smaller argument).

#### Go Implementation:
```go
package main

import "fmt"

func GCD(a, b int) int {
	for b != 0 {
		a, b = b, a%b
	}
	return a
}

func main() {
	fmt.Println("GCD of 48 and 18 is:", GCD(48, 18)) // Output: 6
}
```

---

### B. The Extended Euclidean Algorithm
- **Goal**: Finds the integer coefficients $x$ and $y$ that solve **Bézout's Identity**:
  $$ax + by = \gcd(a, b)$$
- **Enterprise Application (Modular Multiplicative Inverse)**: 
  Used heavily in RSA cryptography to calculate the private key $d$ from the public exponent $e$ and modulus $\phi(n)$, such that:
  $$e \cdot d \equiv 1 \pmod{\phi(n)}$$

#### Go Implementation of Modular Inverse:
```go
package main

import (
	"errors"
	"fmt"
)

// ExtendedGCD returns (gcd, x, y) such that ax + by = gcd
func ExtendedGCD(a, b int) (int, int, int) {
	if b == 0 {
		return a, 1, 0
	}
	gcd, x1, y1 := ExtendedGCD(b, a%b)
	
	// Backtrack values
	x := y1
	y := x1 - (a/b)*y1
	return gcd, x, y
}

// ModularInverse calculates x such that (a * x) % m == 1
func ModularInverse(a, m int) (int, error) {
	gcd, x, _ := ExtendedGCD(a, m)
	if gcd != 1 {
		return 0, errors.New("modular inverse does not exist (a and m are not coprime)")
	}
	// x can be negative, wrap around positive modulo
	return (x%m + m) % m, nil
}

func main() {
	// Find modular inverse of 3 modulo 11 (3 * 4 % 11 == 1)
	inv, err := ModularInverse(3, 11)
	if err != nil {
		fmt.Println("Error:", err)
	} else {
		fmt.Println("Modular Inverse of 3 modulo 11 is:", inv) // Output: 4
	}
}
```

---

## 5. Popular Interview Questions & High-Impact Answers (Extended)

### Q4: Explain the Euclidean Algorithm's time complexity. Why is it highly performant?
* **Answer**: The Euclidean Algorithm operates in **$O(\log \min(a, b))$ time**. By Lamé's Theorem, the number of division steps is guaranteed to be at most **5 times the number of digits** in the smaller integer. It is highly performant because each recursive modulo step (`a % b`) halves the value of the arguments on average. In the worst case (the Fibonacci sequence where successive numbers have golden-ratio relationships), the values decrease by a factor of the golden ratio $\phi \approx 1.618$ at each step, yielding a strict logarithmic complexity bound.

### Q5: What is the "Extended Euclidean Algorithm", and what is its primary use in modern backend security/cryptography?
* **Answer**: The **Extended Euclidean Algorithm** is an extension of the standard GCD calculation that additionally computes integers $x$ and $y$ satisfying Bézout's identity: $ax + by = \gcd(a, b)$. Its primary use in modern backend security and cryptography is to calculate the **Modular Multiplicative Inverse** of a number. This is the cornerstone of **RSA public-key cryptography**, where the algorithm is used to calculate the private key $d$ satisfying $e \cdot d \equiv 1 \pmod{\phi(n)}$, ensuring that data encrypted with the public exponent $e$ can only be decrypted by the holder of the mathematically derived private key $d$.
