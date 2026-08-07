# Backend — Skills Consolidadas

Skills reunidas: **supabase-postgres-best-practices** e **api-endpoint-builder**. Cobrem boas práticas de performance/otimização em Postgres (Supabase) e construção de endpoints REST prontos para produção.

## Índice

- [supabase-postgres-best-practices](#supabase-postgres-best-practices)
- [api-endpoint-builder](#api-endpoint-builder)

---

## supabase-postgres-best-practices

> **Descrição original:** Postgres performance optimization and best practices from Supabase. Use this skill when writing, reviewing, or optimizing Postgres queries, schema designs, or database configurations.
>
> **Fonte:** `supabase/agent-skills`

### When to Use

Use this skill when you need postgres performance optimization and best practices from Supabase. Use this skill when writing, reviewing, or optimizing Postgres queries, schema designs, or database configurations.


Comprehensive performance optimization guide for Postgres, maintained by Supabase. Contains rules across 8 categories, prioritized by impact to guide automated query optimization and schema design.

### When to Apply

Reference these guidelines when:
- Writing SQL queries or designing schemas
- Implementing indexes or query optimization
- Reviewing database performance issues
- Configuring connection pooling or scaling
- Optimizing for Postgres-specific features
- Working with Row-Level Security (RLS)

### Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Query Performance | CRITICAL | `query-` |
| 2 | Connection Management | CRITICAL | `conn-` |
| 3 | Security & RLS | CRITICAL | `security-` |
| 4 | Schema Design | HIGH | `schema-` |
| 5 | Concurrency & Locking | MEDIUM-HIGH | `lock-` |
| 6 | Data Access Patterns | MEDIUM | `data-` |
| 7 | Monitoring & Diagnostics | LOW-MEDIUM | `monitor-` |
| 8 | Advanced Features | LOW | `advanced-` |

### How to Use

Read individual rule files for detailed explanations and SQL examples:

```
references/query-missing-indexes.md
references/query-partial-indexes.md
references/_sections.md
```

Each rule file contains:
- Brief explanation of why it matters
- Incorrect SQL example with explanation
- Correct SQL example with explanation
- Optional EXPLAIN output or metrics
- Additional context and references
- Supabase-specific notes (when applicable)

### References

- https://www.postgresql.org/docs/current/
- https://supabase.com/docs
- https://wiki.postgresql.org/wiki/Performance_Optimization
- https://supabase.com/docs/guides/database/overview
- https://supabase.com/docs/guides/auth/row-level-security

### Limitations

- Use this skill only when the task clearly matches its upstream product or API scope.
- Verify commands, API behavior, pricing, quotas, credentials, and deployment effects against current official documentation before making changes.
- Do not treat generated examples as a substitute for environment-specific tests, security review, or user approval for destructive or costly actions.

---

## api-endpoint-builder

> **Descrição original:** Builds production-ready REST API endpoints with validation, error handling, authentication, and documentation. Follows best practices for security and scalability.
>
> **Fonte:** `community`

Build complete, production-ready REST API endpoints with proper validation, error handling, authentication, and documentation.

### When to Use This Skill

- User asks to "create an API endpoint" or "build a REST API"
- Building new backend features
- Adding endpoints to existing APIs
- User mentions "API", "endpoint", "route", or "REST"
- Creating CRUD operations

### What You'll Build

For each endpoint, you create:
- Route handler with proper HTTP method
- Input validation (request body, params, query)
- Authentication/authorization checks
- Business logic
- Error handling
- Response formatting
- API documentation
- Tests (if requested)

### Endpoint Structure

#### 1. Route Definition

```javascript
// Express example
router.post('/api/users', authenticate, validateUser, createUser);

// Fastify example
fastify.post('/api/users', {
  preHandler: [authenticate],
  schema: userSchema
}, createUser);
```

#### 2. Input Validation

Always validate before processing:

```javascript
const validateUser = (req, res, next) => {
  const { email, name, password } = req.body;
  
  if (!email || !email.includes('@')) {
    return res.status(400).json({ error: 'Valid email required' });
  }
  
  if (!name || name.length < 2) {
    return res.status(400).json({ error: 'Name must be at least 2 characters' });
  }
  
  if (!password || password.length < 8) {
    return res.status(400).json({ error: 'Password must be at least 8 characters' });
  }
  
  next();
};
```

#### 3. Handler Implementation

```javascript
const createUser = async (req, res) => {
  try {
    const { email, name, password } = req.body;
    
    // Check if user exists
    const existing = await db.users.findOne({ email });
    if (existing) {
      return res.status(409).json({ error: 'User already exists' });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    // Create user
    const user = await db.users.create({
      email,
      name,
      password: hashedPassword,
      createdAt: new Date()
    });
    
    // Don't return password
    const { password: _, ...userWithoutPassword } = user;
    
    res.status(201).json({
      success: true,
      data: userWithoutPassword
    });
    
  } catch (error) {
    console.error('Create user error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
};
```

### Best Practices

#### HTTP Status Codes
- `200` - Success (GET, PUT, PATCH)
- `201` - Created (POST)
- `204` - No Content (DELETE)
- `400` - Bad Request (validation failed)
- `401` - Unauthorized (not authenticated)
- `403` - Forbidden (not authorized)
- `404` - Not Found
- `409` - Conflict (duplicate)
- `500` - Internal Server Error

#### Response Format

Consistent structure:

```javascript
// Success
{
  "success": true,
  "data": { ... }
}

// Error
{
  "error": "Error message",
  "details": { ... } // optional
}

// List with pagination
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

#### Security Checklist

- [ ] Authentication required for protected routes
- [ ] Authorization checks (user owns resource)
- [ ] Input validation on all fields
- [ ] SQL injection prevention (use parameterized queries)
- [ ] Rate limiting on public endpoints
- [ ] No sensitive data in responses (passwords, tokens)
- [ ] CORS configured properly
- [ ] Request size limits set

#### Error Handling

```javascript
// Centralized error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  
  // Don't leak error details in production
  const message = process.env.NODE_ENV === 'production' 
    ? 'Internal server error' 
    : err.message;
  
  res.status(err.status || 500).json({ error: message });
});
```

### Common Patterns

#### CRUD Operations

```javascript
// Create
POST /api/resources
Body: { name, description }

// Read (list)
GET /api/resources?page=1&limit=20

// Read (single)
GET /api/resources/:id

// Update
PUT /api/resources/:id
Body: { name, description }

// Delete
DELETE /api/resources/:id
```

#### Pagination

```javascript
const getResources = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;
  
  const [resources, total] = await Promise.all([
    db.resources.find().skip(skip).limit(limit),
    db.resources.countDocuments()
  ]);
  
  res.json({
    success: true,
    data: resources,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
};
```

#### Filtering & Sorting

```javascript
const getResources = async (req, res) => {
  const { status, sort = '-createdAt' } = req.query;
  
  const filter = {};
  if (status) filter.status = status;
  
  const resources = await db.resources
    .find(filter)
    .sort(sort)
    .limit(20);
  
  res.json({ success: true, data: resources });
};
```

### Documentation Template

```javascript
/**
 * @route POST /api/users
 * @desc Create a new user
 * @access Public
 * 
 * @body {string} email - User email (required)
 * @body {string} name - User name (required)
 * @body {string} password - Password, min 8 chars (required)
 * 
 * @returns {201} User created successfully
 * @returns {400} Validation error
 * @returns {409} User already exists
 * @returns {500} Server error
 * 
 * @example
 * POST /api/users
 * {
 *   "email": "user@example.com",
 *   "name": "John Doe",
 *   "password": "securepass123"
 * }
 */
```

### Testing Example

```javascript
describe('POST /api/users', () => {
  it('should create a new user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({
        email: 'test@example.com',
        name: 'Test User',
        password: 'password123'
      });
    
    expect(response.status).toBe(201);
    expect(response.body.success).toBe(true);
    expect(response.body.data.email).toBe('test@example.com');
    expect(response.body.data.password).toBeUndefined();
  });
  
  it('should reject invalid email', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({
        email: 'invalid',
        name: 'Test User',
        password: 'password123'
      });
    
    expect(response.status).toBe(400);
    expect(response.body.error).toContain('email');
  });
});
```

### Key Principles

- Validate all inputs before processing
- Use proper HTTP status codes
- Handle errors gracefully
- Never expose sensitive data
- Keep responses consistent
- Add authentication where needed
- Document your endpoints
- Write tests for critical paths

### Related Skills

- `@security-auditor` - Security review
- `@test-driven-development` - Testing
- `@database-design` - Data modeling

### Limitations
- Use this skill only when the task clearly matches the scope described above.
- Do not treat the output as a substitute for environment-specific validation, testing, or expert review.
- Stop and ask for clarification if required inputs, permissions, safety boundaries, or success criteria are missing.

---
