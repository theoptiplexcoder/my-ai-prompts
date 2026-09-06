# Performance Audit: Login + Database Fetching in a Multi-Tenant Application

You are a senior backend/database performance engineer.

Audit the application's **login flow and database-fetching performance**. The current problem is that login takes too long and pages take too long to fetch data after authentication.

Do NOT immediately rewrite code.

First understand the existing architecture, authentication flow, multi-tenancy model, database schema, Prisma/ORM usage, API routes, server components, client components, middleware, and data-fetching patterns.

Your goal is to identify the actual bottlenecks and then make targeted performance improvements without breaking tenant isolation or authorization.

---

## 1. First: Map the Complete Login Request Path

Trace the login flow from browser → frontend → authentication provider → session/token validation → tenant resolution → database queries → redirect → initial dashboard/page data.

Document every operation in order.

For example:

```text
Browser
  ↓
Login form
  ↓
Supabase Auth / authentication API
  ↓
Session/JWT creation
  ↓
Middleware
  ↓
User lookup
  ↓
Tenant/institution lookup
  ↓
Role lookup
  ↓
Dashboard API
  ↓
Dashboard database queries
  ↓
Response
```

Measure how long each stage takes.

Do not assume authentication itself is slow.

Determine whether the actual delay comes from:

* authentication provider
* middleware
* JWT/session validation
* user lookup
* tenant lookup
* role/permission lookup
* database connection establishment
* database query execution
* multiple sequential queries
* API request overhead
* server-side rendering
* client-side fetching
* duplicate requests
* frontend hydration
* unnecessary redirects

---

# 2. Add Temporary Performance Instrumentation

Before optimizing, add development-only timing instrumentation.

Every important request should report:

```text
request start
auth start/end
tenant resolution start/end
database query start/end
API handler start/end
serialization time
total request time
```

For database queries, capture:

```text
query name
duration
number of rows returned
number of columns selected
whether query used an index
```

Do NOT log:

* passwords
* access tokens
* refresh tokens
* JWT contents
* personal sensitive data
* secrets
* database credentials

Use request IDs so a complete request can be traced.

Example:

```text
request_id=abc123

auth:              180ms
user lookup:        95ms
tenant lookup:      80ms
permissions:        72ms
dashboard query:   210ms
serialization:      15ms
-------------------------
total:             652ms
```

---

# 3. Multi-Tenancy Audit

Treat tenant isolation as a HARD security requirement.

Identify exactly how the application determines the current tenant/institution.

Determine whether the application uses:

* tenant ID in JWT
* user → tenant relationship
* subdomain
* URL parameter
* session metadata
* request headers
* middleware
* database lookup
* some combination

Prefer a design where the authenticated identity can determine the tenant without requiring unnecessary database queries.

For example, investigate whether this inefficient pattern exists:

```text
JWT
 ↓
get user
 ↓
query user table
 ↓
get institution_id
 ↓
query institution
 ↓
get role
 ↓
query role
 ↓
query dashboard
```

Determine whether some of this information can safely be derived from the authenticated session/JWT or fetched in a single query.

However:

**Never put sensitive authorization information into a JWT merely for performance.**

Do not weaken tenant isolation to reduce query count.

---

# 4. Verify Tenant Scoping on EVERY Database Query

Audit every query involved in login and dashboard loading.

Look for queries such as:

```ts
prisma.student.findMany()
```

when they should be scoped like:

```ts
prisma.student.findMany({
  where: {
    institutionId,
    ...
  }
})
```

Every tenant-owned entity must have an explicit tenant boundary unless the schema guarantees tenant isolation through another mechanism.

Check:

* students
* faculty
* courses
* programs
* departments
* attendance
* marks
* timetable
* certificates
* documents
* notifications
* appointments
* users
* academic records
* audit logs

Make sure a request from Tenant A can NEVER retrieve Tenant B's records.

Performance optimization must not remove tenant filters.

---

# 5. Analyze Database Indexes

Inspect the actual database schema and identify the fields used frequently in:

```text
WHERE
JOIN
ORDER BY
GROUP BY
UNIQUE
FOREIGN KEY
```

Pay particular attention to multi-tenant queries.

For example, if queries commonly look like:

```sql
WHERE institution_id = ?
AND status = ?
```

consider whether a composite index is appropriate:

```sql
(institution_id, status)
```

If queries commonly look like:

```sql
WHERE institution_id = ?
AND course_id = ?
AND student_id = ?
```

investigate an appropriate composite index.

Do NOT blindly create indexes for every column.

For every proposed index explain:

1. Which query benefits?
2. Why the existing index is insufficient?
3. Expected performance benefit?
4. Write/storage cost?
5. Is the index actually used?

---

# 6. Check User/Tenant Lookup Specifically

The login path must be extremely efficient.

Find the queries used to retrieve:

```text
authenticated user
tenant/institution
role
permissions
profile
```

Investigate whether these can be consolidated.

For example, instead of:

```text
Query 1:
SELECT user

Query 2:
SELECT institution

Query 3:
SELECT role

Query 4:
SELECT permissions
```

determine whether a properly indexed relational query can retrieve the required information efficiently.

Avoid unnecessary round trips.

If Prisma is being used, inspect whether nested selects/includes are generating inefficient SQL.

Prefer explicit `select` over fetching entire records.

---

# 7. Detect N+1 Queries

Search the codebase for N+1 patterns.

Example of a bad pattern:

```ts
const students = await prisma.student.findMany();

for (const student of students) {
  const attendance = await prisma.attendance.findMany({
    where: {
      studentId: student.id
    }
  });
}
```

This creates:

```text
1 + N queries
```

Replace with an appropriate:

* relation query
* batch query
* aggregation
* grouped query
* join
* `in` query

Example:

```ts
const studentIds = students.map(s => s.id);

const attendance = await prisma.attendance.findMany({
  where: {
    studentId: {
      in: studentIds
    }
  }
});
```

Then group the results in memory when appropriate.

---

# 8. Detect Sequential Database Queries

Search for patterns like:

```ts
const user = await getUser();

const tenant = await getTenant();

const permissions = await getPermissions();

const dashboard = await getDashboard();
```

Determine which operations are independent.

If independent, consider:

```ts
const [user, tenant, permissions, dashboard] =
  await Promise.all([
    getUser(),
    getTenant(),
    getPermissions(),
    getDashboard()
  ]);
```

Only parallelize queries when:

* they are actually independent
* they don't create excessive DB load
* they belong to the same request
* connection pool capacity is sufficient

Do NOT blindly replace every sequential query with `Promise.all()`.

---

# 9. Eliminate Over-Fetching

Audit every `findMany`, `findUnique`, `findFirst`, raw SQL query, and API response.

Look for:

```ts
await prisma.student.findMany()
```

returning entire student records when the UI only needs:

```text
id
name
rollNumber
photo
status
```

Prefer:

```ts
select: {
  id: true,
  name: true,
  rollNumber: true,
  photoUrl: true,
  status: true
}
```

Do this throughout the application.

Pay special attention to:

* dashboard queries
* student lists
* faculty lists
* timetable
* notifications
* certificates
* marks
* attendance
* admin tables

Never return unnecessary large fields such as:

* document blobs
* large JSON objects
* transcripts
* audit metadata
* unnecessary relations

---

# 10. Avoid Unnecessary `include`

Audit Prisma queries containing large nested:

```ts
include: {
  ...
}
```

Determine whether the UI actually needs every included relation.

Replace broad includes with targeted selects where possible.

Bad:

```ts
include: {
  students: true,
  courses: true,
  faculty: true,
  departments: true,
  documents: true,
}
```

if the dashboard only needs counts.

Instead consider:

```text
COUNT()
aggregations
small summary queries
```

rather than loading thousands of records.

---

# 11. Dashboard Query Audit

After login, identify every query required to render the first authenticated page.

Create a table:

| Query       | Purpose      | Rows | Duration | Tenant Scoped | Indexed | Necessary |
| ----------- | ------------ | ---: | -------: | ------------- | ------- | --------- |
| User        | Auth context |    1 |        ? | Yes           | Yes     | Yes       |
| Institution | Tenant       |    1 |        ? | Yes           | Yes     | Yes       |
| Students    | Dashboard    |    ? |        ? | Yes           | ?       | Yes       |
| Faculty     | Dashboard    |    ? |        ? | Yes           | ?       | Yes       |
| Attendance  | Metrics      |    ? |        ? | Yes           | ?       | Yes       |

Find queries that can be:

* removed
* merged
* parallelized
* cached
* aggregated
* deferred
* paginated

---

# 12. Do NOT Fetch Entire Tables for Dashboard Metrics

Look for patterns such as:

```ts
const students = await prisma.student.findMany();
const count = students.length;
```

Replace with a database-side count:

```ts
const count = await prisma.student.count({
  where: {
    institutionId
  }
});
```

Likewise investigate:

```text
count
sum
average
groupBy
exists
distinct
```

instead of transferring unnecessary rows from PostgreSQL to the application server.

---

# 13. Pagination

Audit every potentially large list.

Do not allow:

```ts
findMany()
```

to return an unbounded number of records.

Use pagination.

For large datasets prefer cursor/keyset pagination where appropriate.

Example concept:

```text
WHERE institution_id = ?
AND id > ?
ORDER BY id
LIMIT 50
```

rather than repeatedly using large offsets:

```text
OFFSET 100000
```

Use offset pagination only where it is appropriate and dataset sizes are manageable.

---

# 14. Query Ordering

Inspect queries containing:

```sql
ORDER BY
```

especially when combined with:

```sql
WHERE institution_id = ?
LIMIT ?
```

Determine whether indexes can support the filtering and ordering.

Example:

```sql
WHERE institution_id = ?
ORDER BY created_at DESC
LIMIT 20
```

may benefit from an appropriate composite index.

Do not add an index without checking the actual query workload.

---

# 15. Query Plan Analysis

For the slowest queries, run:

```sql
EXPLAIN ANALYZE
```

or the database equivalent.

Look for:

* sequential scans
* expensive joins
* large row estimates
* bad cardinality estimates
* unnecessary sorting
* nested loop problems
* missing indexes
* excessive rows scanned
* excessive rows returned

For each slow query provide:

```text
Query
↓
Execution plan
↓
Root cause
↓
Recommended fix
↓
Expected impact
```

Do not optimize based solely on intuition.

---

# 16. Connection Pooling

Investigate database connection management.

Determine whether every request creates a new Prisma/database client.

Bad:

```ts
const prisma = new PrismaClient();
```

inside request handlers/modules that are repeatedly instantiated.

Check for:

* connection exhaustion
* excessive connection creation
* connection pool limits
* serverless connection behavior
* Prisma client lifecycle
* Supabase/Postgres connection pooling
* PgBouncer if applicable

The solution must match the deployment environment.

---

# 17. Serverless / Vercel / Render Considerations

Determine where the application runs.

If running on serverless infrastructure:

* minimize database round trips
* use appropriate connection pooling
* avoid creating excessive connections
* avoid huge queries
* keep server-side operations efficient

If running on a persistent server:

* verify connection pool configuration
* monitor pool utilization
* avoid unnecessarily opening connections

Do not introduce Redis merely to hide inefficient database queries.

First optimize:

```text
query design
indexes
tenant filtering
connection management
caching strategy
API architecture
```

---

# 18. Authentication Optimization

Determine which authentication calls happen on:

* middleware
* server layout
* page
* API route
* client component

Look for duplicate authentication checks.

Example:

```text
Middleware → getUser()
Layout → getUser()
Page → getUser()
API → getUser()
```

This can create repeated work.

Design a clear authentication boundary.

The application should not repeatedly perform the same expensive auth/database operation during one request unless required for security.

---

# 19. Server Components vs Client Fetching

If using Next.js App Router, inspect whether authenticated data is unnecessarily fetched from the browser.

Bad pattern:

```text
Browser
 ↓
Page loads
 ↓
JavaScript executes
 ↓
fetch('/api/user')
 ↓
fetch('/api/tenant')
 ↓
fetch('/api/dashboard')
```

when the data could safely be fetched on the server during rendering.

Determine whether Server Components can directly call the backend/data-access layer.

Avoid unnecessary:

```text
Server → API route → database
```

when the request is already executing on the server and can safely use:

```text
Server Component → data access → database
```

Do not bypass security boundaries where they are genuinely required.

---

# 20. Caching Strategy

Identify data that changes rarely.

Potential candidates:

```text
institution branding
program metadata
course metadata
academic configuration
static lookup tables
permissions/configuration
```

Determine what can be cached safely.

For tenant-specific data, cache keys MUST include tenant identity.

Bad:

```text
dashboard
```

Good:

```text
tenant:{tenantId}:dashboard
```

Never allow cached Tenant A data to be returned to Tenant B.

Do not cache highly dynamic or authorization-sensitive data without carefully validating invalidation and isolation.

---

# 21. Database-Level Multi-Tenancy Safety

Determine whether PostgreSQL Row Level Security is being used.

If RLS is used:

* verify policies
* verify tenant context
* verify indexes
* verify that policies do not cause unnecessary performance problems

If RLS is not used:

* identify the application's tenant-isolation boundary
* verify every query explicitly applies tenant filtering
* identify possible accidental cross-tenant access

Do not disable tenant protections for performance.

Security comes first.

---

# 22. API Response Optimization

Measure response size.

Check whether APIs return:

```text
large objects
unused fields
nested relations
duplicate data
```

Use DTOs / serializers / explicit response shapes.

For dashboards, return only the information required by the UI.

---

# 23. Frontend Performance

After database optimization, measure:

```text
TTFB
server response
API latency
JS execution
hydration
rendering
```

Determine whether the perceived slowness is actually frontend-related.

Do not blame PostgreSQL without evidence.

---

# 24. Create a Performance Baseline

Before making changes, record:

```text
Login total time
Authentication time
First authenticated page TTFB
Dashboard database time
Number of DB queries
Total DB rows returned
Response size
```

Run each important test multiple times.

Record:

```text
p50
p95
p99
```

where possible.

One unusually fast request is not sufficient evidence.

---

# 25. Load Testing

Test multiple tenants and concurrent users.

At minimum simulate:

```text
1 user
5 users
20 users
50 users
100 users
```

where the environment can safely handle it.

Test:

```text
login
dashboard loading
student listing
student profile
attendance
marks
timetable
certificate listing
```

Do not test only a single tenant.

Use multiple tenants with different dataset sizes.

Example:

```text
Tenant A: 100 students
Tenant B: 1,000 students
Tenant C: 10,000 students
Tenant D: 100,000 students
```

Determine whether query performance scales with tenant size.

---

# 26. Test Tenant Isolation During Load

Performance testing must also verify correctness.

While testing multiple tenants:

```text
Tenant A requests
Tenant B requests
Tenant C requests
```

verify that:

```text
Tenant A never receives Tenant B data
Tenant B never receives Tenant C data
```

Check:

* API responses
* database queries
* cache
* SSR
* client state
* pagination
* search
* filtering
* exports

---

# 27. Identify the Top 10 Slowest Queries

Produce a final report:

```text
1. Query X — 850ms
   Cause:
   Fix:
   Expected improvement:

2. Query Y — 430ms
   Cause:
   Fix:
   Expected improvement:
```

Rank them by actual impact.

---

# 28. Optimization Priority

Prioritize fixes in this order:

### P0 — Security/Critical

* tenant isolation problems
* authorization bugs
* cross-tenant data leakage
* missing tenant filters

### P1 — Major Performance

* N+1 queries
* sequential independent queries
* missing critical indexes
* unbounded queries
* duplicate authentication queries
* excessive database round trips

### P2 — Important

* over-fetching
* unnecessary relations
* inefficient pagination
* expensive dashboard queries
* unnecessary API layers

### P3 — Optimization

* caching
* response compression
* minor frontend optimizations
* secondary indexes
* serialization improvements

---

# 29. Make Changes Incrementally

Do NOT rewrite the entire data layer.

For each optimization:

1. Record baseline.
2. Change one logical bottleneck.
3. Run tests.
4. Run performance measurement.
5. Compare results.
6. Verify tenant isolation.
7. Keep the change only if it improves the system without introducing correctness/security issues.

---

# 30. Final Deliverable

After auditing the application, provide:

## A. Architecture Findings

Explain the current login/data-fetch flow.

## B. Login Bottleneck

Identify exactly why login is slow.

## C. Database Bottleneck

Identify exactly which queries are slow and why.

## D. Multi-Tenancy Findings

Show how tenant isolation currently works and identify any weaknesses.

## E. Query Findings

List:

* N+1 queries
* duplicate queries
* sequential queries
* over-fetching
* missing pagination
* inefficient joins
* missing indexes
* unnecessary relations
* unnecessary counts/data transfers

## F. Index Recommendations

Provide:

```text
Table
Index
Query it improves
Reason
```

## G. Code Changes

Provide the exact files/functions that should be changed.

Do not modify unrelated code.

## H. Before vs After

Report:

```text
Metric              Before       After
------------------------------------------------
Login               X ms         Y ms
DB queries           X            Y
Dashboard            X ms         Y ms
Rows returned        X            Y
Response size        X KB         Y KB
```

## I. Security Verification

Explicitly verify:

```text
Tenant isolation: PASS/FAIL
Authorization: PASS/FAIL
Cache isolation: PASS/FAIL
Pagination isolation: PASS/FAIL
Search isolation: PASS/FAIL
```

## J. Remaining Bottlenecks

Clearly state what is still slow after optimization.

---

# Important Rules

1. **Do not optimize blindly. Measure first.**
2. **Never sacrifice tenant isolation for performance.**
3. **Never remove authorization checks just to reduce latency.**
4. **Do not introduce Redis unless there is a demonstrated caching/use-case need.**
5. **Do not add indexes blindly.**
6. **Do not fetch entire tables when only counts or summaries are needed.**
7. **Avoid N+1 queries.**
8. **Avoid unnecessary sequential database calls.**
9. **Use explicit SELECT/select fields instead of returning entire records.**
10. **Paginate large datasets.**
11. **Use composite indexes where the actual query pattern justifies them.**
12. **Use database-side aggregation instead of transferring large datasets.**
13. **Avoid duplicate auth/session/database lookups.**
14. **Cache only when the data and invalidation strategy are clearly understood.**
15. **Tenant ID must be part of cache keys for tenant-specific cached data.**
16. **Run EXPLAIN ANALYZE on genuinely slow queries.**
17. **Test with multiple tenants and different tenant dataset sizes.**
18. **Do not declare the problem solved based on one fast request.**
19. **Preserve existing functionality and API contracts unless a change is justified.**
20. **Prefer simple architecture and fewer round trips over adding infrastructure.**

The primary objective is:

> **Make login and authenticated data fetching fast, predictable, scalable, and tenant-safe by fixing the actual database and request-path bottlenecks rather than masking them with unnecessary infrastructure.**
