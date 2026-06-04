# Linked Lists

Comprehensive interview study guide covering Singly vs. Doubly Linked Lists, pointer manipulation, and cycle detection algorithms.

---

## 1. Singly vs. Doubly Linked Lists

A **Linked List** is a linear data structure where elements are not stored contiguously in memory. Instead, each element (node) is an independent object containing data and a **pointer (reference)** pointing to the next node.

| Attribute | Singly Linked List | Doubly Linked List |
| :--- | :--- | :--- |
| **Pointer Overhead** | 1 pointer per node (`next`) | 2 pointers per node (`next`, `prev`) |
| **Traversal Direction**| Forward only | Forward and Backward |
| **Insertion/Deletion** | Easy (if previous node is known) | Easier (no need to find previous node) |
| **Memory Footprint** | Low | Higher (due to extra pointer overhead) |

---

## 2. Core Linked List Operations

### 1. Reversing a Linked List
Reversing a singly linked list requires flipping pointer directions in-place using three pointers (`prev`, `curr`, `next_node`):

```javascript
let prev = null, curr = head;
while (curr !== null) {
    let next_node = curr.next; // Store next node
    curr.next = prev;          // Flip pointer direction
    prev = curr;               // Move prev forward
    curr = next_node;          // Move curr forward
}
return prev; // New head
```
* **Complexity:** $O(n)$ time and $O(1)$ space.

### 2. Cycle Detection: Floyd's Tortoise & Hare Algorithm
Determines if a linked list contains a loop (cycle) using two pointers moving at different speeds:
* **Slow Pointer:** Moves 1 step at a time (`slow = slow.next`).
* **Fast Pointer:** Moves 2 steps at a time (`fast = fast.next.next`).
* **Logic:** If a cycle exists, the fast pointer will eventually catch up and collide with the slow pointer inside the loop. If the fast pointer hits `null`, there is no cycle.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: How do you reverse a Singly Linked List in $O(1)$ auxiliary space?
* **Answer:** Reversing is done iteratively in-place to ensure constant $O(1)$ space. Maintain three pointers: `prev` (initially null), `curr` (initialized to head), and a temporary `next_node`. Loop through the list:
  1. Save the next node: `next_node = curr.next`.
  2. Reverse the current node's pointer: `curr.next = prev`.
  3. Move pointers forward: set `prev = curr`, then set `curr = next_node`.
  * Repeat until `curr` becomes null. `prev` will be pointing to the new head of the reversed list.

### Q2: Explain Floyd's Tortoise and Hare algorithm and how to locate the exact starting node of a cycle.
* **Answer:**
  1. **Detection:** Initialize `slow` and `fast` pointers at the head. Move `slow` by 1 node and `fast` by 2 nodes. If they collide (`slow == fast`), a cycle is proven.
  2. **Locate Start Node:** After collision, leave `slow` at the meeting point and reset `fast` back to the `head` of the list.
  3. Move *both* pointers forward at the same speed of 1 step at a time.
  4. The node where they collide again is the **exact starting node** of the cycle.
  * **Proof:** Mathematical alignment shows the distance from the head to the cycle start is exactly equal to the distance from the first meeting point to the cycle start.

### Q3: Why is Sentinel (Dummy) Node pattern highly useful in Linked List coding questions?
* **Answer:** A **Sentinel (Dummy) Node** is a temporary, empty placeholder node created at the beginning of a linked list (`let dummy = new ListNode(0); dummy.next = head`). It is highly useful because it **eliminates edge cases** associated with modifying the head of the list (such as deleting the head, inserting before the head, or merging two empty lists). Instead of writing complex `if (head == null)` or special pointer reassignments for the first element, you execute standard middle-list pointer swaps on `dummy.next` and return `dummy.next` at the end.
