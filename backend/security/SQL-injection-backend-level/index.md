# SQL Injection (SQLi)

SQL Injection is a vulnerability where an attacker manipulates application queries by inputting untrusted SQL code, bypassing security barriers to access, modify, or destroy database records.

---

## 1. Meaning & Mechanism

SQLi occurs when user inputs are dynamically concatenated directly into database query strings instead of being handled separately.

### Vulnerable Code Example (Go)
```go
// VULNERABLE: Direct string concatenation
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s' AND password = '%s'", email, password)
db.Query(query)
```
If an attacker inputs `admin@example.com' --` for the email, the resulting SQL query becomes:
```sql
SELECT * FROM users WHERE email = 'admin@example.com' --' AND password = ''
```
The double-dash `--` is an SQL comment, ignoring the password check completely and granting unauthorized access.

---

## 2. Prevention & Mitigation Best Practices

### 1. Parameterized Queries / Prepared Statements (Primary Defense)
Parameterized queries separate SQL instructions from user-supplied data.

```
                     PREPARED STATEMENT EXECUTION MODEL
       
Step 1: Parse & Pre-compile (Instruction Definition Phase)
Client ───(Sends SQL Template only)───> DB Engine
          "SELECT * FROM users WHERE email = ?"
                                           │
                                           v  (DB Compiles execution path)
                                    [SQL Query Compiled]
                                           │
Step 2: Parameter Execution (Literal Binding Phase)
Client ───(Sends literal parameters only) ─┼───> [User Input bound safely as data]
          "admin@example.com' OR '1'='1"   │     (Never parsed as SQL code)
                                           v
                                    [Query Safely Executed]
```

* **How it works:**
  1. The server compiles the SQL template with placeholders (`?`, `$1`) first.
  2. The database driver transmits the parameters separately.
  3. The database engine executes the query, treating parameters strictly as literals (plain values) and never as executable code.

#### Secure Code Example (Go)
```go
// SECURE: Parameterized query using placeholders
query := "SELECT id, name FROM users WHERE email = $1 AND password = $2"
db.QueryRow(query, email, password)
```

### 2. Enforce Safe ORMs & Query Builders
Using structured mappers like Hibernate, Prisma, or sqlc ensures queries are parameterized automatically:
* Ensure developers understand that ORMs *can* still be vulnerable if raw SQL query concatenation bypasses standard API calls.

### 3. Handle Dynamic Identifiers Safely (Whitelisting)
Placeholders **cannot** be used for database identifiers such as table names, column names, or sort orders (`ORDER BY`).
* *Mitigation:* Implement strict whitelisting. If users can sort a table, map the user input to a predefined set of columns:
```go
// Whitelisting sorting columns
allowedColumns := map[string]bool{"created_at": true, "name": true}
if !allowedColumns[sortBy] {
    sortBy = "created_at" // Default fallback
}
query := fmt.Sprintf("SELECT * FROM posts ORDER BY %s ASC", sortBy)
```

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: Why do Parameterized Queries completely prevent SQL Injection?
* **Answer:** Parameterized queries (prepared statements) force the database to compile and define the execution plan of the SQL query *before* user input is inserted. When parameters are sent later, the database engine treats them strictly as data/literals. Even if the parameter contains SQL keywords like `DROP TABLE` or `' OR '1'='1`, they are never parsed as SQL commands, neutralizing SQL Injection.

### Q2: Can you perform parameterized queries on database identifiers like table names or column names?
* **Answer:** No, SQL engines do not allow parameterization of table names, column names, or structural SQL keywords. If you need dynamic table or column queries, you must apply strict whitelisting. Match user-supplied strings against a safe, predefined list of identifiers in your code before dynamically injecting them.

### Q3: What is "Blind SQL Injection" and how does it differ from standard SQL Injection?
* **Answer:** In standard SQL Injection, the database returns verbose error messages or records directly on the screen, showing the attacker query results. In **Blind SQL Injection**, the application does not print data. Instead, attackers must infer information by asking the database True/False questions and observing:
  1. **Boolean-based changes:** Does the page layout change or load slightly differently when a query resolves to true?
  2. **Time-based latency:** Does the query execute a sleep command (`pg_sleep()`) if a guessed character is correct, resulting in a delayed response time?
