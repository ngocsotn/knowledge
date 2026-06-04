# Pagination, Filtering, and Sorting Design

Comprehensive study guide for designing robust, scalable, and high-performance search, filtering, sorting, and pagination interfaces for web APIs and relational databases.

---

## 1. Offset-Based vs. Cursor-Based Pagination

When returning large datasets, dividing them into manageable chunks (pages) is critical. There are two primary architectural patterns.

### A. Offset-Based Pagination (Limit / Offset)
Clients request a specific slice of data using `page` or `limit` and `offset` variables.
- **Example API**: `GET /api/posts?limit=10&offset=50`
- **SQL Execution**:
  ```sql
  SELECT * FROM posts ORDER BY created_at DESC LIMIT 10 OFFSET 50;
  ```

#### Performance Bottleneck (The Deep Pagination Problem):
When executing `OFFSET 500000`, the database engine cannot jump directly to row 500,000. It must read, parse, and sort **all 500,010 rows**, discard the first 500,000, and return only the last 10.
- **Complexity**: $O(N)$ where $N$ is the offset depth.
- **Result**: Drastic increase in CPU, disk I/O, and query response latency for high offsets.
- **Data Drifting/Inconsistency**: If rows are inserted or deleted while a user is navigating pages, items can be duplicated or skipped across page boundaries.

---

### B. Cursor-Based Pagination (Keyset / Token)
Clients request data relative to a specific reference element from the previous page (the **cursor**), which is typically a unique, sequential identifier or a timestamp combined with a primary key.
- **Example API**: `GET /api/posts?limit=10&cursor=eyJpZCI6MTIzLCJjcmVhdGVkX2F0IjoiMjAyNi0wNi0wMSJ9` (Base64 encoded JSON)
- **SQL Execution**:
  ```sql
  SELECT * FROM posts 
  WHERE (created_at, id) < ('2026-06-01', 123) 
  ORDER BY created_at DESC, id DESC 
  LIMIT 10;
  ```

#### Why it is High-Performance:
By filtering on the index keys (`created_at`, `id`), the database can perform an index range scan (B-Tree lookup) and jump directly to the target row, reading only the next 10 rows.
- **Complexity**: $O(\log N)$ (index lookup) + $O(\text{limit})$. Constant performance regardless of page depth.
- **Data Drifting Solved**: Since boundaries are absolute values instead of relative index markers, inserts/deletes do not cause duplicate or missed items on navigation.

---

### Architectural Comparison

| Dimension | Offset-Based | Cursor-Based |
| :--- | :--- | :--- |
| **Deep Page Speed** | Extremely Slow (scans all discarded rows) | **Extremely Fast** (constant time lookup) |
| **Data Consistency** | Poor (vulnerable to drifts/updates) | **Excellent** (resilient to database changes) |
| **Direct Page Jumping**| **Supported** (can jump directly to page 54) | Unsupported (can only navigate next/prev) |
| **Implementation** | Trivial | Complex (requires multi-column index keys) |
| **Use Case** | Admin grids, tables with small datasets | Infinite scroll, massive tables (feeds, logs) |

---

## 2. Dynamic Filtering & Sorting Safely

Implementing custom filtering and sorting requires dynamic SQL query construction. This introduces significant security and performance risks if done incorrectly.

### Safe Sorting and SQL Injection Protection
- **Vulnerability**: You **cannot** use SQL bind parameters (placeholders like `?` or `$1`) for column names or ordering directions in `ORDER BY` clauses:
  - ❌ `SELECT * FROM users ORDER BY $1 $2;` -- **Fails or is insecure.**
- **Mitigation Pattern (Whitelisting)**:
  Always strictly validate column inputs against a hardcoded whitelist of allowed sorting fields and directions in the application code before constructing the query:
  ```go
  // Go Whitelisting Example
  allowedSortColumns := map[string]bool{"id": true, "created_at": true, "price": true}
  
  column := r.URL.Query().Get("sort_by")
  direction := strings.ToUpper(r.URL.Query().Get("direction"))
  
  if !allowedSortColumns[column] {
      column = "created_at" // Default fallback
  }
  if direction != "ASC" && direction != "DESC" {
      direction = "DESC" // Default fallback
  }
  
  // Safe to interpolate only after strict whitelisting
  query := fmt.Sprintf("SELECT * FROM products ORDER BY %s %s LIMIT 10", column, direction)
  ```

---

## 3. Hard Interview Questions & Deep Answers

### Q1: How do you design cursor-based pagination when sorting by a non-unique column (e.g., product price)?
**Answer**:
Sorting by a non-unique column (like `price`) is highly problematic for cursors. If multiple products share the exact same price, the system cannot uniquely identify where the last page ended, leading to skipped or duplicated records.
- **The Solution: Multi-Column Keyset Pagination**:
  To guarantee uniqueness, always append a unique secondary column (typically the primary key `id`) to the sort configuration.
  1. **The Compound Index**: Create a composite B-Tree index on `(price, id)`.
  2. **The SQL Filter**: Compare the tuple values using row comparison syntax:
     ```sql
     SELECT * FROM products
     WHERE (price, id) > (19.99, 452)
     ORDER BY price ASC, id ASC
     LIMIT 10;
     ```
  3. **Cursor Creation**: Construct the next cursor by combining both values from the last retrieved record: `price: 19.99, id: 452` (Base64 serialized).

### Q2: How do you handle dynamic search and multi-field filtering on a database table with millions of rows without degrading performance?
**Answer**:
Allowing clients to filter dynamically by combinations of fields (e.g., `status`, `category_id`, `created_at`, `user_id`) presents indexing challenges. You cannot create indexes for every possible permutation of columns.
- **Execution Strategy**:
  1. **Partial/Composite Indexes**: Analyze query telemetry to discover the most common filter combinations. Create composite indexes for those, putting high-cardinality/equality fields first, and range fields last.
  2. **Covering Indexes**: If a query only needs a few fields, design a composite index containing those fields, allowing the database to return results directly from the index tree without a second heap lookup.
  3. **Elasticsearch / OpenSearch Offloading**: If multi-field text search and fuzzy matching are heavily used, offload this traffic from the primary relational database. Replicate the database writes asynchronously (via Outbox Pattern or CDC - Change Data Capture) into a search engine optimized for inverted indexing.

### Q3: Why is offset-based pagination still used despite its scaling drawbacks? When is it the superior option?
**Answer**:
Offset-based pagination is still highly useful in scenarios where:
1. **Total Page Count and Jumping are Required**: Standard admin dashboards or e-commerce pagination grids require a "Jump to Page X" interface, which is mathematically impossible with pure cursor-based pagination (since cursor position depends on reading prior records).
2. **Small/Bounded Datasets**: If the maximum table size is small (e.g., less than 50,000 rows), the overhead of scanning offsets is negligible (milliseconds), and offset pagination is simpler to implement.
3. **Low Concurrency / Static Data**: If data changes rarely (e.g., a list of countries or states), the risk of data drifting (skipping/duplicating rows) during navigation is virtually zero.
