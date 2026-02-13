---
mako: "1.0"
type: docs
entity: "Express.js Middleware Guide"
updated: 2026-01-15
tokens: 210
language: en
summary: "How to write and use middleware in Express.js, with examples"
audience: developers
freshness: monthly

links:
  internal:
    - url: /docs/routing
      context: "Express routing documentation"
      type: sibling
    - url: /docs/error-handling
      context: "Error handling middleware patterns"
      type: child
    - url: /api-reference/app-use
      context: "app.use() API reference"
      type: reference
  external:
    - url: https://expressjs.com/en/guide/using-middleware.html
      context: "Official Express middleware guide"
      type: source

tags:
  - express
  - nodejs
  - middleware
  - backend
---

# Express.js Middleware Guide

## Overview
Middleware functions execute during the request-response cycle. They access the `req`, `res` objects and call `next()` to pass control to the next middleware. Express apps are essentially a series of middleware calls.

## Usage

### Basic middleware
```javascript
app.use((req, res, next) => {
  console.log(`${req.method} ${req.path}`);
  next();
});
```

### Route-specific middleware
```javascript
app.get('/api/users', authenticate, (req, res) => {
  res.json(req.user);
});
```

### Error-handling middleware
```javascript
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Internal server error' });
});
```

## Execution Order
Middleware executes in the order registered via `app.use()`. Error handlers must have 4 parameters and are registered last. Route-specific middleware runs only for matching routes.

## Common Middleware
- `express.json()` — parse JSON request bodies
- `express.static()` — serve static files
- `cors()` — enable Cross-Origin Resource Sharing
- `helmet()` — set security HTTP headers
- `morgan()` — HTTP request logging
- `express-rate-limit` — rate limiting

## See Also
- Error handling patterns: `/docs/error-handling`
- Authentication middleware examples: `/docs/auth`
- Performance middleware (compression, caching): `/docs/performance`
