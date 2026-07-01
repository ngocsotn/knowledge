# SaaS Multi-Tenant Architecture

Multi-tenancy is an architecture where a single physical deployment of a software application serves multiple distinct enterprise customers (Tenants).
- **Isolation Levels:**
  1. **Shared App, Separate DB:** Highest isolation; high database cost.
  2. **Shared App, Shared DB, Separate Schema:** Balanced isolation and cost.
  3. **Shared App, Shared DB, Shared Schema:** Row-Level Security (RLS) gating; highest density; lowest cost; highest risk of data leaks.

## Interview Questions & Answers

### Q1: How do you enforce data isolation in a shared-database multi-tenant schema?
- **Answer:** Enable PostgreSQL **Row-Level Security (RLS)**. Configure RLS policies on tables to automatically filter rows based on the current tenant context variable (e.g., `SET LOCAL app.current_tenant = 'tenant_id'`), rendering cross-tenant queries impossible even if a developer forgets a `WHERE tenant_id` clause.
