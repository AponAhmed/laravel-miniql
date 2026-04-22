# MiniQL — Laravel GraphQL-Lite Query Engine

[![PHP](https://img.shields.io/badge/PHP-8.1%2B-blue)](https://www.php.net)
[![Laravel](https://img.shields.io/badge/Laravel-10%2B%20|%2011%2B-red)](https://laravel.com)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

A **production-grade, single-endpoint internal query engine** for Laravel.  
Think GraphQL — without the schema language, without the AST parser, with full Laravel idioms.

---

## Table of Contents

1. [What is MiniQL?](#what-is-miniql)
2. [Installation](#installation)
3. [Quick Start](#quick-start)
4. [Query Syntax Reference](#query-syntax-reference)
5. [Mutation Syntax Reference](#mutation-syntax-reference)
6. [Schema Config Reference](#schema-config-reference)
7. [Resolvers](#resolvers)
8. [Mutation Handlers](#mutation-handlers)
9. [Hooks](#hooks)
10. [Fluent PHP API](#fluent-php-api)
11. [Caching](#caching)
12. [Security](#security)
13. [Introspection](#introspection)
14. [Artisan Commands](#artisan-commands)
15. [Architecture Overview](#architecture-overview)
16. [Upgrading / Roadmap](#roadmap)

---

## What is MiniQL?

MiniQL lets your frontend (or internal services) send expressive queries to a **single POST endpoint** instead of building dozens of REST routes.

```
POST /api/miniql
```

A request looks like this:

```json
{
  "query": {
    "users": {
      "where":    { "active": true },
      "fields":   ["id", "name", "email"],
      "relations": {
        "posts": {
          "fields":  ["id", "title"],
          "orderBy": { "column": "created_at", "direction": "desc" },
          "limit":   5
        }
      },
      "orderBy": { "column": "name", "direction": "asc" },
      "page":    1,
      "perPage": 20
    }
  }
}
```

**Everything you send is validated against a whitelist schema. No raw SQL exposure. No N+1.**

---

## Installation

```bash
composer require yourvendor/miniql
```

Publish the config:

```bash
php artisan vendor:publish --tag=miniql-config
```

---

## Quick Start

### 1. Register your models in `config/miniql.php`

```php
'models' => [

    'users' => [
        'model'     => App\Models\User::class,
        'fields'    => ['id', 'name', 'email', 'created_at'],
        'relations' => ['posts'],
        'mutations' => [
            'createUser' => App\MiniQL\Mutations\CreateUserMutation::class,
            'updateUser' => App\MiniQL\Mutations\UpdateUserMutation::class,
        ],
    ],

    'posts' => [
        'model'     => App\Models\Post::class,
        'fields'    => ['id', 'title', 'body', 'user_id'],
        'relations' => ['user'],
        'mutations' => [
            'createPost' => App\MiniQL\Mutations\CreatePostMutation::class,
        ],
    ],

],
```

### 2. Send a query

```bash
curl -X POST http://yourapp.test/api/miniql \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "users": {
        "fields": ["id", "name"],
        "limit": 5
      }
    }
  }'
```

**Response:**

```json
{
  "data": {
    "users": [
      { "id": 1, "name": "Alice" },
      { "id": 2, "name": "Bob" }
    ]
  },
  "errors": []
}
```

---

## Query Syntax Reference

Every query node supports:

| Key           | Type              | Description                                              |
|---------------|-------------------|----------------------------------------------------------|
| `fields`      | `string[]`        | Fields to return (whitelist enforced)                    |
| `where`       | `object`          | Simple equality OR advanced `{op, value}` filter         |
| `whereIn`     | `object`          | Column → array of values                                 |
| `whereNull`   | `string[]`        | Columns that must be NULL                                |
| `whereNotNull`| `string[]`        | Columns that must NOT be NULL                            |
| `search`      | `object`          | Full-text LIKE search `{term, fields[]}`                 |
| `orderBy`     | `string\|object`  | String column, or `{column, direction}`                  |
| `limit`       | `int`             | Max rows (hard-capped by `security.max_results`)         |
| `page`        | `int`             | Enable pagination (returns `{data, meta}`)               |
| `perPage`     | `int`             | Rows per page (default from config)                      |
| `relations`   | `object`          | Eager-load nested relations (recursively supported)      |

### Advanced `where` example

```json
{
  "query": {
    "users": {
      "where": {
        "age":    { "op": ">=", "value": 18 },
        "status": "active"
      },
      "whereIn":     { "role": ["admin", "editor"] },
      "whereNotNull": ["email_verified_at"],
      "search": { "term": "alice", "fields": ["name", "email"] }
    }
  }
}
```

### Pagination example

```json
{
  "query": {
    "posts": {
      "fields":  ["id", "title", "created_at"],
      "orderBy": { "column": "created_at", "direction": "desc" },
      "page":    2,
      "perPage": 10
    }
  }
}
```

**Response shape with pagination:**

```json
{
  "data": {
    "posts": {
      "data": [ ... ],
      "meta": {
        "current_page": 2,
        "per_page": 10,
        "total": 87,
        "last_page": 9,
        "from": 11,
        "to": 20
      }
    }
  },
  "errors": []
}
```

### Multi-type query (batch)

```json
{
  "query": {
    "users": { "fields": ["id", "name"], "limit": 3 },
    "posts": { "fields": ["id", "title"], "limit": 3 }
  }
}
```

Both are resolved in a single request. No N+1. Each type runs its own eager-load strategy.

---

## Mutation Syntax Reference

```json
{
  "mutation": {
    "createUser": {
      "data": {
        "name":     "Apon",
        "email":    "apon@example.com",
        "password": "secret123"
      }
    }
  }
}
```

- All mutations run inside a **database transaction**. If any mutation fails, all are rolled back.
- Multiple mutations can be sent in one request.

```json
{
  "mutation": {
    "createUser": { "data": { "name": "Bob", "email": "bob@x.com", "password": "pass1234" } },
    "createPost": { "data": { "title": "Hello World", "user_id": 1 } }
  }
}
```

### Query + Mutation in one request

```json
{
  "mutation": {
    "updateUser": { "id": 3, "data": { "name": "Updated" } }
  },
  "query": {
    "users": { "where": { "id": 3 }, "fields": ["id", "name"] }
  }
}
```

Mutations run first, then queries — so the query returns the updated state.

---

## Schema Config Reference

Full annotated config (published to `config/miniql.php`):

```php
'models' => [
    'users' => [

        // ✅ Required: Eloquent model class
        'model' => App\Models\User::class,

        // ✅ Required: whitelisted selectable fields
        'fields' => ['id', 'name', 'email', 'created_at'],

        // Whitelisted eager-loadable relation names
        'relations' => ['posts', 'profile'],

        // Custom resolver (overrides default query builder)
        'resolver' => App\MiniQL\Resolvers\UserResolver::class,

        // Mutation handlers, keyed by mutation name
        'mutations' => [
            'createUser' => App\MiniQL\Mutations\CreateUserMutation::class,
            'updateUser' => App\MiniQL\Mutations\UpdateUserMutation::class,
            'deleteUser' => App\MiniQL\Mutations\DeleteUserMutation::class,
        ],

        // Before/after hooks for query and mutation lifecycle
        'hooks' => [
            'before_query'    => App\MiniQL\Hooks\UserBeforeQueryHook::class,
            'after_query'     => App\MiniQL\Hooks\UserAfterQueryHook::class,
            'before_mutation' => null,
            'after_mutation'  => null,
        ],

        // Global Eloquent scopes applied to every query on this type
        // e.g. 'active' calls ->active() on the query builder
        'scopes' => ['active'],

        // Free-form metadata (documentation, versioning, etc.)
        'meta' => ['description' => 'Registered users'],
    ],
],
```

---

## Resolvers

A Resolver replaces the default query builder for a specific type. Use it for:

- Multi-tenant scoping
- Auth-based filtering
- Complex joins that can't be expressed in `where`

### Generate a resolver

```bash
php artisan miniql:make-resolver UserResolver
```

### Implement it

```php
// app/MiniQL/Resolvers/UserResolver.php

namespace App\MiniQL\Resolvers;

use MiniQL\Resolvers\BaseResolver;
use Illuminate\Database\Eloquent\Builder;

class UserResolver extends BaseResolver
{
    protected function model(): string
    {
        return \App\Models\User::class;
    }

    public function query(array $node): Builder
    {
        $q = parent::query($node); // applies basic where filters

        // Scope to current tenant
        $q->where('tenant_id', auth()->user()->tenant_id);

        return $q;
    }
}
```

### Register it

```php
// config/miniql.php
'users' => [
    'resolver' => App\MiniQL\Resolvers\UserResolver::class,
    ...
],
```

---

## Mutation Handlers

### Generate a mutation

```bash
php artisan miniql:make-mutation CreateUser
```

### Implement it

```php
// app/MiniQL/Mutations/CreateUserMutation.php

namespace App\MiniQL\Mutations;

use MiniQL\Mutations\BaseMutation;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

class CreateUserMutation extends BaseMutation
{
    protected function rules(): array
    {
        return [
            'data.name'     => 'required|string|max:255',
            'data.email'    => 'required|email|unique:users,email',
            'data.password' => 'required|string|min:8',
        ];
    }

    public function handle(array $node): mixed
    {
        $data = $this->validate($node); // throws ValidationException on failure

        return User::create([
            'name'     => $data['name'],
            'email'    => $data['email'],
            'password' => Hash::make($data['password']),
        ]);
    }
}
```

`$this->validate($node)` runs Laravel's Validator against `rules()`. On failure it throws a `ValidationException` that is caught by the controller and returned in the `errors` key.

---

## Hooks

Hooks let you tap into the query/mutation lifecycle without modifying the engine.

| Hook              | Receives          | Called                          |
|-------------------|-------------------|---------------------------------|
| `before_query`    | `&$node` (array)  | Before the query executes       |
| `after_query`     | `&$result` (mixed)| After query result is built     |
| `before_mutation` | `&$node` (array)  | Before mutation handler runs    |
| `after_mutation`  | `&$result` (mixed)| After mutation handler returns  |

```php
// app/MiniQL/Hooks/AuditAfterMutationHook.php

namespace App\MiniQL\Hooks;

use MiniQL\Contracts\HookInterface;

class AuditAfterMutationHook implements HookInterface
{
    public function handle(mixed &$context): void
    {
        logger()->info('[Audit] Mutation completed.', ['result' => $context]);
    }
}
```

Register in config:

```php
'hooks' => [
    'after_mutation' => App\MiniQL\Hooks\AuditAfterMutationHook::class,
],
```

---

## Fluent PHP API

Use MiniQL directly in PHP (controllers, jobs, services) without HTTP:

```php
use MiniQL\Facades\MiniQL;

// Simple
$users = MiniQL::query('users')
    ->fields(['id', 'name', 'email'])
    ->where('active', true)
    ->whereIn('role', ['admin', 'editor'])
    ->orderBy('name', 'asc')
    ->limit(50)
    ->get();

// Paginated
$result = MiniQL::query('posts')
    ->fields(['id', 'title', 'created_at'])
    ->with('user', ['fields' => ['id', 'name']])
    ->orderBy('created_at', 'desc')
    ->paginate(15, 1)
    ->get();
// $result['data'], $result['meta']

// Full payload
$result = MiniQL::execute([
    'query' => [
        'users' => ['fields' => ['id', 'name'], 'limit' => 5],
    ],
    'mutation' => [
        'createUser' => ['data' => ['name' => 'Eve', 'email' => 'eve@x.com', 'password' => 'pass1234']],
    ],
]);
```

---

## Caching

Enable Redis-backed query caching in `.env`:

```
MINIQL_CACHE=true
MINIQL_CACHE_TTL=120
MINIQL_CACHE_STORE=redis
```

Cache keys are derived from `type + md5(node)` — identical queries return cached results instantly.

Cache is **automatically invalidated** per-type when a mutation runs (if `cache.auto_invalidate = true`).

---

## Security

All security options live in `config/miniql.php` under `security`:

| Option           | Default | Description                                      |
|------------------|---------|--------------------------------------------------|
| `max_depth`      | `5`     | Max relation nesting depth                       |
| `max_query_nodes`| `10`    | Max top-level query types per request            |
| `max_results`    | `1000`  | Hard cap on rows returned                        |
| `require_auth`   | `false` | Set to `true` to require authentication globally |
| `rate_limit`     | `60`    | Requests per minute per IP (0 = disabled)        |

**Field & relation whitelisting** is always enforced — there is no way to query a field not listed in `config('miniql.models.*.fields')`.

**Protect the endpoint** by adding `auth:sanctum` (or any Laravel middleware) to the route middleware:

```php
// config/miniql.php
'route' => [
    'middleware' => ['api', 'auth:sanctum'],
],
```

---

## Introspection

```bash
GET /api/miniql/schema
```

Returns the full registered schema as JSON (useful for frontend tooling):

```json
{
  "schema": {
    "users": {
      "fields":    ["id", "name", "email", "created_at"],
      "relations": ["posts"],
      "mutations": ["createUser", "updateUser", "deleteUser"],
      "meta":      {}
    }
  },
  "version": "1.0"
}
```

Disable in production:

```env
MINIQL_INTROSPECTION=false
```

Or protect with middleware:

```php
'introspection' => [
    'enabled'    => true,
    'middleware' => ['auth:sanctum'],
],
```

---

## Artisan Commands

| Command                       | Description                                              |
|-------------------------------|----------------------------------------------------------|
| `miniql:make-resolver Name`   | Scaffold a resolver class                                |
| `miniql:make-mutation Name`   | Scaffold a mutation handler class                        |
| `miniql:schema-dump`          | Pretty-print the registered schema                       |
| `miniql:schema-dump --json`   | Output schema as raw JSON                                |
| `miniql:schema-validate`      | Verify all model/resolver/mutation classes exist on disk |

---

## Architecture Overview

```
POST /api/miniql
       │
       ▼
MiniQLController
       │
       ├─ SchemaValidator       ← whitelist check (fields, relations, depth)
       │
       ├─ QueryEngine
       │      ├─ ResolverInterface (custom) OR default Eloquent builder
       │      ├─ Eager loading  ← N+1 prevention via ->with()
       │      ├─ QueryCache     ← Redis-backed result cache
       │      └─ Hooks          ← before/after lifecycle
       │
       └─ MutationEngine
              ├─ DB::transaction ← atomic multi-mutation
              ├─ BaseMutation    ← Laravel validation built-in
              └─ Hooks           ← before/after lifecycle
```

---

## Roadmap

Possible future upgrades:

- [ ] **Query complexity scoring** (prevent expensive queries by weight)
- [ ] **String DSL parser** — accept `users { id name posts { title } }` syntax
- [ ] **WebSocket subscriptions** — real-time updates on mutations
- [ ] **Persisted queries** — hash-indexed pre-registered queries
- [ ] **Automatic SQL optimization** — detect and merge redundant joins
- [ ] **OpenAPI export** — generate an OpenAPI 3 spec from the schema config

---

## License

MIT © Muhiminul Haque
