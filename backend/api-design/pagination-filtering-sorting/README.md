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

## 2. CTE-Based Pagination & Deferred Joins

When scaling relational databases, standard offset pagination or complex filtering combined with massive table joins can lead to extreme performance degradation. Utilizing **Common Table Expressions (CTEs)** and **Deferred Joins** enables ultra-high-performance pagination.

### A. Deferred Joins / Late Row Lookups
A deferred join is an optimization where the database filters and paginates only the primary keys/IDs first, then joins the result back to the same table (or related tables) to fetch the wider columns and heavy payloads.

#### The Problem:
If a query contains joins or selects large text/binary columns (e.g., `body`, `metadata_json`), standard offset pagination loads all columns of all scanned rows into memory before discarding those skipped by the offset.
```sql
-- SLOW: Loads heavy text columns for 50,010 rows, joins with users, then throws away 50,000 rows
SELECT p.id, p.title, p.body, u.username 
FROM posts p
JOIN users u ON p.user_id = u.id
ORDER BY p.created_at DESC 
LIMIT 10 OFFSET 50000;
```

#### The Solution (Deferred Join via CTE):
We paginate only on the lightweight indexed keys in a CTE to get the 10 target IDs, then join back to load the heavy data columns.
```sql
WITH paginated_keys AS (
    SELECT id 
    FROM posts 
    ORDER BY created_at DESC 
    LIMIT 10 OFFSET 50000
)
SELECT p.id, p.title, p.body, u.username 
FROM posts p
JOIN paginated_keys pk ON p.id = pk.id
JOIN users u ON p.user_id = u.id
ORDER BY p.created_at DESC;
```
- **Why it is fast**: The CTE `paginated_keys` only scans the lightweight B-Tree index on `(created_at, id)` to fetch 10 IDs. The database only performs the heavy `JOIN` on the final 10 rows instead of 50,010 rows, avoiding loading megabytes of unneeded text and joining user records that are discarded.

---

### B. Single-Query Total Count + Paginated Rows (Combined CTE)
Typically, APIs returning paginated data require a total count for UI pagination elements. This normally forces two database round-trips:
1. `SELECT COUNT(*) FROM posts WHERE status = 'published';`
2. `SELECT * FROM posts WHERE status = 'published' LIMIT 10 OFFSET 50;`

Under high load, executing two separate scans on large tables doubles database overhead. A CTE can combine both tasks into a **single SQL query execution**.

#### The Combined CTE Query:
```sql
WITH filtered_posts AS (
    SELECT id, title, created_at
    FROM posts
    WHERE status = 'published'
),
total_count AS (
    SELECT COUNT(*) AS total FROM filtered_posts
),
paginated_rows AS (
    SELECT id, title, created_at
    FROM filtered_posts
    ORDER BY created_at DESC
    LIMIT 10 OFFSET 50
)
SELECT 
    p.id, 
    p.title, 
    p.created_at, 
    c.total
FROM paginated_rows p
CROSS JOIN total_count c;
```

#### How to Parse in Go:
```go
type PaginatedPost struct {
    ID        int64     `db:"id"`
    Title     string    `db:"title"`
    CreatedAt time.Time `db:"created_at"`
    TotalCount int64    `db:"total"` // Captured from the CROSS JOIN
}

func FetchPaginatedPosts(db *sql.DB, limit, offset int) ([]PaginatedPost, int64, error) {
    query := `
        WITH filtered_posts AS (
            SELECT id, title, created_at FROM posts WHERE status = 'published'
        ),
        total_count AS (
            SELECT COUNT(*) AS total FROM filtered_posts
        ),
        paginated_rows AS (
            SELECT id, title, created_at FROM filtered_posts
            ORDER BY created_at DESC LIMIT $1 OFFSET $2
        )
        SELECT p.id, p.title, p.created_at, c.total
        FROM paginated_rows p
        CROSS JOIN total_count c;`

    rows, err := db.Query(query, limit, offset)
    if err != nil {
        return nil, 0, err
    }
    defer rows.Close()

    var posts []PaginatedPost
    var total int64

    for rows.Next() {
        var p PaginatedPost
        if err := rows.Scan(&p.ID, &p.Title, &p.CreatedAt, &p.TotalCount); err != nil {
            return nil, 0, err
        }
        posts = append(posts, p)
        total = p.TotalCount // Same for all rows
    }

    return posts, total, nil
}
```

#### Performance Trade-off (The CTE Optimization Fence):
- **PostgreSQL < 12**: CTEs were strict **optimization fences**. The engine evaluated the entire CTE in memory before applying outer filters, causing full table scans even if the outer query had a LIMIT.
- **PostgreSQL 12+**: The optimizer dynamically inlines CTEs (treated as subqueries) unless they are explicitly marked as `MATERIALIZED`.
- **Best Practice**: If the count portion is extremely heavy, caching the total count in Redis or utilizing an approximate count (`SELECT reltuples FROM pg_class`) is preferred over executing a full count in every query.

---

## 3. Dynamic Filtering & Sorting Safely

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

## 4. Advanced Indexing Rules for Complex Filters

When constructing indexes to support dynamic filter combinations, understanding B-Tree index properties is critical.

### The Equality-Range Rule (Index Order Rule)
When designing a composite index to support filters with both equality and range conditions (e.g., `WHERE status = 'active' AND price > 100`):
- **Rule**: Place **equality** columns *first* in the index definition, and **range/inequality** columns *last*.
- **Index Definition**: `CREATE INDEX idx_products_status_price ON products (status, price);`
- **Why**: 
  - The database first navigates the index tree directly to the `status = 'active'` branch (an exact B-Tree node lookup).
  - From there, it performs a sequential, linear range scan across contiguous nodes where `price > 100`.
  - If you created the index as `(price, status)`, filtering by `price > 100` forces the database to scan the entire index segment for values greater than 100, checking the status condition on each index node, leading to a much larger index scan volume.

### Index Scan vs. Index Intersection vs. Bitmap Index Scan
- **Index Scan (Composite Index)**: Multiple columns merged into a single multi-column B-Tree. Perfect for matching queries that use all or a prefix of those columns.
- **Index Intersection (Bitmap Index Scan)**: Occurs when the database planner uses two separate, single-column indexes on two different fields (e.g., an index on `category_id` and another on `user_id`). The planner scans both indexes, builds memory bitmaps of qualifying row IDs, intersects the bitmaps (using `AND`/`OR` operations), and then fetches only the intersecting row headers from the main heap.
- **Trade-off**: Composite indexes are significantly faster and consume less CPU than dynamically intersecting separate indexes, but they require storing larger, dedicated index structures.

---

## 5. Scalable Full-Text Search (FTS)

Standard pattern matching using `LIKE '%pattern%'` is a common performance bottleneck:
- `LIKE '%pattern%'` (leading wildcard) **cannot use standard B-Tree indexes**. It forces the database to execute a full table scan, parsing and pattern-matching every single string on every row.
- `LIKE 'pattern%'` (trailing wildcard only) can use B-Tree indexes but fails to find matches inside words.

### PostgreSQL Native Full-Text Search (TSVector and GIN)
For massive tables where offloading to Elasticsearch is too heavy, native FTS using a **GIN (Generalized Inverted Index)** is the production-ready alternative.

1. **How it works**: Words are parsed into lexemes (normalized root words, e.g., "jumping" and "jumped" map to root "jump") using dictionaries.
2. **Schema setup**:
   ```sql
   -- Create a generated column that automatically keeps a normalized TSVector of text fields
   ALTER TABLE posts ADD COLUMN search_vector tsvector
     GENERATED ALWAYS AS (
       to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))
     ) STORED;
   ```
3. **The Index**: Use a **GIN index** on the `search_vector` column:
   ```sql
   CREATE INDEX idx_posts_search_vector ON posts USING gin(search_vector);
   ```
4. **Execution**:
   ```sql
   -- Perform instant inverted index lookup using @@ operator
   SELECT title, body, ts_rank(search_vector, query) as rank
   FROM posts, to_tsquery('english', 'database & scaling') as query
   WHERE search_vector @@ query
   ORDER BY rank DESC;
   ```
   - **Performance**: Instant millisecond lookups regardless of row counts, since the GIN index directly points to the specific document IDs containing the searched lexemes.

---

## 6. Hard Interview Questions & Deep Answers

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

