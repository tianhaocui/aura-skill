---
name: aura-framework
description: Teaches AI how to build backend services using the Aura Java framework. Auto-triggers when user mentions Aura, or when working in a project with aura-web dependency.
version: 0.1.0
triggers:
  - aura
  - aura framework
  - aura-web
  - io.aura
---

# Aura Framework Skill

## What is Aura

Aura is an AI-native lightweight Java 17+ backend framework. It prioritizes minimal code, zero configuration, and automatic MCP tool generation.

## When to Use This Skill

- User asks to create a backend service with Aura
- Project has `aura-web` in pom.xml dependencies
- User mentions "Aura framework" or imports from `io.aura`

## Core Principles

1. **One file = one service** — A complete app is a single main method
2. **No annotations required** — Service methods are plain Java
3. **No config files required** — Code is configuration
4. **Return value = response** — Non-void methods auto-serialize to JSON
5. **Parameters auto-bind** — int/String from path/query, record from body

## Minimal App Pattern

```java
// Simplest: one-line startup, auto-scans caller's package for @Path classes
public class App {
    public static void main(String[] args) {
        Aura.run(args);
    }
}
```

```java
// With configuration
public class App {
    public static void main(String[] args) {
        Aura.create()
            .port(8080)
            .cors(true)
            .mcp(true)
            .scan("com.example")
            .start(args);
    }
}
```

## Route Registration Patterns

```java
// Pattern 1: @Path class + auto-scan (preferred for AI)
@Path("/user")
public class UserService {
    public User get(int id) { ... }       // GET /user/{id}
    public List<User> list() { ... }      // GET /user
    public User create(CreateReq req) { ... } // POST /user
    public void delete(int id) { ... }    // DELETE /user/{id}
}
// Registered automatically by Aura.run() or Aura.scan()

// Pattern 2: Manual routes (for custom endpoints)
r.get("/health", ctx -> ctx.text("ok"));
r.get("/user/{id}", userService, "get");
r.crud("/user", userService);  // 5 routes in 1 line
```

## Service Class Pattern

Service classes are plain Java. No annotations, no interfaces, no framework types in signatures:

```java
public class UserService {
    private final Db db;
    public UserService(Db db) { this.db = db; }

    public Row get(int id) { return db.findById("user", id); }
    public List<Row> list() { return db.table("user").find(); }
    public Row create(CreateReq req) {
        Validate.notBlank(req.name(), "name required");
        return Row.of("user").set("name", req.name()).insert(db);
    }
    public void delete(int id) { db.deleteById("user", id); }
}

record CreateReq(String name, int age) {}
```

## Parameter Binding Rules

| Parameter type | Source | Example |
|---------------|--------|---------|
| int, long, String | path param first, then query | `int id` → from `{id}` or `?id=` |
| record / POJO | request body (JSON) | `CreateReq req` → deserialized |
| Context | framework context | for advanced use only |

## Database Pattern

```java
Db db = Db.create("jdbc:mysql://localhost/mydb", "user", "pass");

// Dynamic SQL — null/blank params auto-skipped (recommended for complex queries)
String sql = "SELECT * FROM user #where(name, '=', name) #and(age, '>', age) #orderBy(created)";
db.find(sql, filterMap);                        // List<Row>
db.paginate(sql, filterMap, pageNum, pageSize); // Page<Row>

// Query builder (simple CRUD shortcut)
db.table("user").where("age", ">", 18).find();
db.table("user").where("id", 1).findOne();

// Row CRUD
Row.of("user").set("name", "tom").set("age", 25).insert(db);
Row.of("user").id(123).set("name", "new").update(db);

// Shortcuts
db.findById("user", id);
db.deleteById("user", id);

// Transaction
db.transaction(() -> { db.execute(sql, args); });
```

## Middleware Pattern

```java
r.before(ctx -> { /* auth, logging */ });
r.after(ctx -> { /* timing */ });
r.group("/api", api -> {
    api.before(authMiddleware);
    api.get("/items", itemService, "list");
});
r.exception(Validate.ValidationException.class, (e, ctx) ->
    ctx.status(400).json(Map.of("error", e.getMessage())));
```

## MCP Integration

```java
Aura.create()
    .mcp(true)  // enables MCP Server on port+1
    .routes(r -> r.crud("/user", userService))
    .start(args);
// --mcp-stdio flag for Claude Desktop/Cursor integration
```

## Testing Pattern

```java
var test = TestClient.of(app);
test.get("/user/1").expect(200).bodyContains("user-1");
test.post("/user").body(new CreateReq("tom", 25)).expect(200);
test.get("/notfound").expect(404);
```

## Configuration

```java
Aura.create()
    .port(8080)              // or env: AURA_PORT
    .env("dev")              // or env: AURA_ENV
    .cors(true)              // CORS
    .mcp(true)               // MCP Server
    .prop("db.url", "...")   // custom props (env: DB_URL)
    .onStart(a -> a.register(db))
    .onStop(a -> db.close())
    .start(args);            // --port=N --env=X --mcp-stdio
```

## Third-party Integration Pattern

Aura has no DI container. Use `register/get` to share objects across services:

```java
// Register at startup
Db db = Db.create("jdbc:mysql://localhost/mydb", "user", "pass");
RedissonClient redis = Redisson.create(config);

Aura.create()
    .register(db)
    .register(redis)
    .onStop(a -> a.get(RedissonClient.class).shutdown())
    .service(new CacheService(redis))
    .service(new UserService(db))
    .start(args);
```

```java
// Service receives dependencies via constructor — explicit, no magic
public class CacheService {
    private final RedissonClient redis;

    public CacheService(RedissonClient redis) {
        this.redis = redis;
    }

    @Get("/{key}")
    public Object get(String key) {
        return redis.getBucket(key).get();
    }
}
```

```java
// If you need app itself (e.g. to access props or multiple registered objects)
public class MyService {
    private final RedissonClient redis;
    private final Db db;

    public MyService(Aura app) {
        this.redis = app.get(RedissonClient.class);
        this.db = app.get(Db.class);
    }
}
```

Key rules:
- Always pass dependencies via constructor, never use static fields
- Register shared objects with `app.register()`, retrieve with `app.get(Type.class)`
- Clean up resources with `app.onStop()`
- Do NOT build a DI graph — if wiring gets complex, just pass objects directly



- Do NOT use `@Autowired` or DI — just `new` your services
- Do NOT create config files — use code + env vars
- Do NOT add Spring dependencies — Aura is standalone
- Do NOT write `ctx.path()` in Service methods — let framework bind params
- Do NOT forget `start(args)` — use args version for --mcp-stdio support

## Maven Dependency

```xml
<parent>
    <groupId>io.github.tianhaocui</groupId>
    <artifactId>aura-parent</artifactId>
    <version>0.1.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>io.github.tianhaocui</groupId>
        <artifactId>aura-web</artifactId>
    </dependency>
    <!-- Optional -->
    <dependency>
        <groupId>io.github.tianhaocui</groupId>
        <artifactId>aura-db</artifactId>
    </dependency>
</dependencies>
```

Always inherit `aura-parent` — it provides `-parameters` compiler flag required for parameter name binding.
