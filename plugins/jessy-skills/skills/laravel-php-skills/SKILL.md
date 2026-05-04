---
name: laravel-php-skills
description: "MUST use when working on any Laravel PHP project, writing or reviewing Laravel code, creating models/controllers/migrations/policies/jobs/form requests/services/Eloquent queries, writing or fixing Pest tests, styling Blade templates with Tailwind CSS, or when user mentions 'laravel', 'artisan', 'eloquent', 'pest', 'blade', 'tailwind', 'migration', 'controller', 'model', 'queue', or 'livewire'. Also triggers on: N+1 queries, caching, authorization, validation, routing, scheduling, API resources, Vite errors, or any PHP 8.3+ backend code in a Laravel context."
---

# Laravel PHP Skills — Best Practices Skill
*Full-stack Laravel skill covering backend patterns, Pest testing, and Tailwind CSS v4*

## Activation

When this skill activates, output:

`⚡ Laravel PHP Skills active — applying Laravel best practices, Pest testing patterns, and Tailwind v4 conventions.`

Then execute the protocol below.

## Context Guard

| Context | Status |
|---------|--------|
| **Laravel PHP backend code** | ACTIVE — full protocol |
| **Pest tests (Feature/Unit/Browser)** | ACTIVE — full protocol |
| **Blade templates with Tailwind** | ACTIVE — full protocol |
| **Spring Boot / Java** | DORMANT — use springboot-restful skill |
| **Vanilla PHP without Laravel** | DORMANT — do not activate |

---

## Stack Versions (Remote Signing Portal)

- PHP 8.3, Laravel v13, Tailwind v4, Pest v4, PHPUnit v12
- laravel/boost v2, laravel/pint v1, laravel/pail v1

---

## Part 1 — Laravel Best Practices

### Consistency First

Before applying any rule, check what the application already does. The best choice is the one the codebase already uses. Check sibling files, related controllers, models, or tests for established patterns — follow them, don't introduce a second way.

---

### Artisan Commands

- Use `php artisan make:` for ALL new files; always pass `--no-interaction`
- New models: also create factory + seeder (`php artisan make:model --help` for options)
- `php artisan route:list` — inspect routes; filter: `--method`, `--name`, `--path`
- `php artisan config:show app.name` — read config values
- Read `.env` directly for environment variables
- Tinker: always single-quoted shell strings, double-quoted PHP strings inside

---

### Database Performance

- Eager load with `with()` to prevent N+1 queries
- Enable `Model::preventLazyLoading()` in development (AppServiceProvider)
- Select only needed columns — avoid `SELECT *`
- Use `chunk()` / `chunkById()` for large datasets
- Index columns used in `WHERE`, `ORDER BY`, `JOIN`
- Use `withCount()` instead of loading relations just to count
- Use `cursor()` for memory-efficient read-only iteration (no relationships)
- Use `lazy()` when you need relationships — chunked, flat LazyCollection
- Use `lazyById()` when updating records while iterating (safe against mutation)
- Never query inside Blade templates

**Pagination strategy:**
- Use `->paginate()` for server-side filtering/sorting (controller handles query params)
- Use `->get()` when client-side JS (KTDataTable, DataTables) handles filtering/pagination
- Always chain `->withQueryString()` on `paginate()` so filter params survive page links

---

### Advanced Query Patterns

**Subquery instead of eager-loading for a single has-many value:**
```php
public function scopeWithLastLoginAt($query): void
{
    $query->addSelect([
        'last_login_at' => Login::select('created_at')
            ->whereColumn('user_id', 'users.id')
            ->latest()->take(1),
    ])->withCasts(['last_login_at' => 'datetime']);
}
```

**`whereIn` + subquery over `whereHas` (index-friendly, no PHP memory):**
```php
// Incorrect — correlated EXISTS re-executes per row
$query->whereHas('company', fn ($q) => $q->where('name', 'like', $term));

// Correct — index lookup
$query->whereIn('company_id', Company::where('name', 'like', $term)->select('id'));
```

**Conditional aggregates — one query instead of N count queries:**
```php
$statuses = Feature::toBase()
    ->selectRaw("count(case when status = 'Requested' then 1 end) as requested")
    ->selectRaw("count(case when status = 'Planned' then 1 end) as planned")
    ->first();
```

**Prevent circular N+1 with `setRelation()`:**
```php
$feature->load('comments.user');
$feature->comments->each->setRelation('feature', $feature);
```

**Compound indexes for multi-column `orderBy`** — index column order must match `ORDER BY` order.

---

### Security

- Define `$fillable` or `$guarded` on every model
- Authorize every action via policies or gates
- No raw SQL with user input — use Eloquent or query builder
- Use `{{ }}` for output escaping in Blade
- Add `@csrf` on all POST/PUT/DELETE forms
- Apply `throttle` middleware on auth and API routes
- Validate MIME type, extension, and size for file uploads
- Never commit `.env`; use `config()` for secrets; use `encrypted` cast for sensitive DB fields
- Pass data to JS via `data-*` attributes or `@json()` — never raw interpolation in `<script>` tags

---

### Caching

```php
// Cache-aside — use remember, not manual get/put
$val = Cache::remember('stats', 60, fn () => $this->computeStats());

// High-traffic — stale-while-revalidate (fresh 5 min, stale served up to 10 min)
Cache::flexible('users', [300, 600], fn () => User::all());

// Per-request memoization — avoids redundant cache hits within one request
Cache::memo()->get('settings');

// Pure in-memory memoization (no cache store hit)
public function roles(): Collection
{
    return once(fn () => $this->loadRoles());
}

// Atomic conditional write — no race condition
Cache::add('lock', true, 10);

// Cache tags — flush related group atomically (redis/memcached only)
Cache::tags(['user-1'])->flush();

// Race condition — use lock
Cache::lock('process-'.$id, 10)->block(5, function () use ($order) {
    $order->process();
});
```

---

### Eloquent Patterns

**Relationships — always return type-hinted:**
```php
public function comments(): HasMany { return $this->hasMany(Comment::class); }
public function author(): BelongsTo { return $this->belongsTo(User::class, 'user_id'); }
```

**Local scopes for reusable query constraints:**
```php
public function scopeActive(Builder $query): Builder
{
    return $query->where('verified', true)->whereNotNull('activated_at');
}
// Usage: User::active()->get()
```

**Global scopes sparingly** — they silently modify every query; prefer local scopes and opt-in.

**Attribute casts — always define in `casts()` method:**
```php
protected function casts(): array
{
    return ['is_active' => 'boolean', 'metadata' => 'array', 'ordered_at' => 'datetime'];
}
```

**`whereBelongsTo()` over manual FK queries:**
```php
Post::whereBelongsTo($user)->get();
Post::whereBelongsTo($user, 'author')->get();
```

**Never hardcode table names** — use Eloquent or `(new Model)->getTable()` in raw queries. Exception: migrations (frozen snapshots).

---

### Routing & Controllers

- Implicit route model binding — type-hint models in controller methods, not `findOrFail($id)`
- Scoped bindings for nested resources: `->scopeBindings()`
- Use `Route::resource()` or `apiResource()` for RESTful endpoints
- Keep controller methods under 10 lines — extract to Action or Service classes
- Type-hint Form Requests for auto-validation and authorization before method executes
- Use named routes and `route()` helper for all URL generation
- Group routes with shared middleware, prefix, and name
- API routes: use versioning (`/api/v1/`) and Eloquent API Resources

---

### Validation

- Use Form Request classes — never validate inline in controllers
- Always use `$request->validated()` — never `$request->all()`
- Array notation preferred: `['required', 'email', Rule::unique('users')]`
- `Rule::when()` for conditional validation
- `after()` method for cross-field custom validation (not `withValidator()`)
- `bail` to stop on first failure

---

### Architecture

**Action classes — one class per business operation:**
```php
class CreateOrderAction {
    public function __construct(private InventoryService $inventory) {}
    public function execute(array $data): Order { ... }
}
```

**Dependency injection — constructor only, no `app()` or `resolve()` inside classes.**

**`defer()` for lightweight post-response work** (logging, analytics) — runs after HTTP response, no queue overhead. Use jobs when work must survive crashes or needs retry.

**`Context` facade for request-scoped data** — propagates through middleware, controllers, jobs, and log entries automatically.

**`Concurrency::run()` for parallel independent operations:**
```php
[$users, $orders] = Concurrency::run([
    fn () => User::count(),
    fn () => Order::where('status', 'pending')->count(),
]);
```

**Convention over configuration** — don't override `$table`, `$primaryKey`, relationship table names unless genuinely non-standard.

**`mb_*` or `Str::` helpers** over raw PHP string functions (UTF-8 safety).

**Default sort** — always `->latest()` or explicit `ORDER BY` — never rely on undefined row order.

---

### Collections

- Higher-order messages: `$users->each->markAsVip()` instead of closure
- `cursor()` — one model in memory, no relationship support; `lazy()` — chunked, supports relationships
- `lazyById()` when updating records during iteration (offset-safe)
- `$users->toQuery()->update([...])` — bulk operations without manual `whereIn`
- `#[CollectedBy(UserCollection::class)]` — declarative custom collection class

---

### HTTP Client

```php
// Always set explicit timeouts
Http::timeout(5)->connectTimeout(3)->get('https://api.example.com/users');

// Retry with backoff for external APIs
Http::retry([100, 500, 1000])->timeout(10)->post('https://api.stripe.com/v1/charges', $data);

// Always check status — Http doesn't throw on 4xx/5xx by default
Http::timeout(5)->get('https://api.example.com/users/1')->throw();

// Concurrent independent requests
$responses = Http::pool(fn (Pool $pool) => [
    $pool->as('users')->get('.../users'),
    $pool->as('posts')->get('.../posts'),
]);

// In tests — always fake, never hit real APIs
Http::preventStrayRequests();
Http::fake(['api.example.com/*' => Http::response([...])]);
```

---

### Queue & Jobs

- Always specify the queue connection and name explicitly
- Use `ShouldBeUnique` to prevent duplicate jobs; `ShouldBeUniqueUntilProcessing` for early lock release
- `retry_after` must exceed job `timeout`; use exponential backoff `[1, 5, 10]`
- Always implement `failed()`; with `retryUntil()`, set `$tries = 0`
- Test jobs with `Queue::fake()`

---

### Task Scheduling

- `withoutOverlapping()` — prevents second instance when first hasn't finished
- `onOneServer()` — required on multi-server deployments (needs shared cache driver)
- `runInBackground()` — concurrent long tasks instead of sequential
- `environments(['production'])` — restrict production-only tasks
- `takeUntilTimeout()` — time-bound unbounded cursors to prevent overlap
- Schedule groups for shared config (timezone, server):
```php
Schedule::daily()->onOneServer()->timezone('Asia/Kuala_Lumpur')->group(function () {
    Schedule::command('reports:generate');
    Schedule::command('reports:prune');
});
```

---

### Migrations

- Use `php artisan make:migration` always — never create migration files manually
- `$table->foreignId('user_id')->constrained()->cascadeOnDelete()` — `constrained()` for all FKs
- Never modify migrations that have already run in production — create a new migration
- Add indexes at creation time (columns used in WHERE, ORDER BY, JOIN)
- Mirror DB column defaults in model `$attributes`
- Write reversible `down()` by default; forward-fix migration for intentionally irreversible changes
- One concern per migration — never mix DDL (schema) and DML (data)

---

### Blade & Views

- `$attributes->merge(['class' => '...'])` in components — don't hardcode classes
- `@pushOnce` for per-component scripts inside `@foreach` (prevents duplicate script injection)
- Blade components over `@include` — explicit props, no implicit variable leakage
- View Composers for data shared across multiple views
- `->fragmentIf($request->hasHeader('HX-Request'), 'fragment-name')` for htmx/Turbo partial re-renders
- `@aware` for deeply nested component props (avoids prop drilling)
- Pass data to JS via `@json()` on data attributes, not inside `<script>` string interpolation:
```blade
<div id="clients-data" data-clients='@json($clients)'></div>
```

---

### Client-Side JS Integration (Blade + KTDataTable)

When using KTDataTable (Metronic) for client-side filtering/pagination:
- Controller returns `->get()` — one DB query, all data
- Table wrapper: `data-kt-datatable="true" data-kt-datatable-state-save="false"`
- Table element: `data-kt-datatable-table="true"`
- Search input: `data-kt-datatable-search="#table_id"`
- Per-page select: `data-kt-datatable-size="true"`
- Pagination div: `data-kt-datatable-pagination="true"`
- Info span: `data-kt-datatable-info="true"`
- Get instance for custom JS hooks: `KTDataTable.getInstance(tableEl)`
- Column filter API: `datatable.setFilter({ column: N, type: 'text', value }).redraw()`
- Global search API: `datatable.search(value)`
- Wrap page-specific JS in `@push('scripts')` at bottom of view

---

### Error Handling

- Use typed custom exceptions
- Handle in global exception handler (`bootstrap/app.php` or exception class `render()`/`report()`)
- `ShouldntReport` for exceptions that should never log
- Return consistent JSON error responses for APIs

---

### Code Conventions

- Descriptive names: `isRegisteredForDiscounts` not `discount()`
- Prefer shorter Laravel syntax: `session('cart')`, `back()`, `now()`, `->latest()`, `->value('name')`
- Reuse existing components before writing new ones
- Stick to existing directory structure — no new base folders without approval
- No dependency changes without approval
- Only create documentation files if explicitly requested

**Laravel naming conventions:**

| Entity | Convention | Example |
|--------|-----------|---------|
| Controller | Singular | `ArticleController` |
| Model | Singular | `User` |
| Table | Plural snake_case | `article_comments` |
| Column | snake_case, no model prefix | `meta_title` |
| Foreign key | singular model + `_id` | `article_id` |
| Route name | snake_case dots | `users.show_active` |
| Variable | camelCase | `$activeUsers` |
| View file | kebab-case | `show-filtered.blade.php` |
| Enum key | TitleCase | `AdminClient`, `Monthly` |

**Use Laravel helpers over raw PHP:**
- Strings: `Str::slug()`, `Str::limit()`, `Str::of()`, `Str::contains()`, `Str::uuid()`
- Arrays: `Arr::get()`, `Arr::only()`, `Arr::first()`, `Arr::wrap()`
- Numbers: `Number::format()`, `Number::currency()`, `Number::abbreviate()`
- URLs: `Uri::of()->withQuery([...])`

---

### PHP 8.3 Standards

- Always use curly braces for control structures, even single-line
- Constructor property promotion: `public function __construct(public Foo $foo) {}`
- Explicit return types and type hints on ALL methods
- TitleCase Enum keys: `FavoritePerson`, `Monthly`
- PHPDoc blocks preferred over inline comments; array shape types in PHPDoc
- Only inline comments for exceptionally complex logic

---

### After Editing PHP Files

Always run: `vendor/bin/pint --dirty --format agent`
Never use `--test` flag — just run and fix.

### Frontend Bundling

If UI changes aren't reflected: ask user to run `npm run build`, `npm run dev`, or `composer run dev`.
Vite manifest error → run `npm run build`.

---

## Part 2 — Pest Testing (Laravel)

### Creating Tests

```bash
php artisan make:test --pest {name}          # Feature test
php artisan make:test --pest --unit {name}   # Unit test
```

Most tests should be feature tests. Do NOT delete tests without approval.

### Running Tests

```bash
php artisan test --compact                          # All tests
php artisan test --compact --filter=testName        # Filtered
php artisan test --compact tests/Feature/FooTest.php
```

### Test Structure

Check existing test files first — use `test()` if the project uses `test()`, `it()` if it uses `it()`.

```php
it('creates a user', function () {
    $user = User::factory()->create();
    expect($user->name)->not->toBeEmpty();
});
```

### Assertions — Use Specific over Generic

| Use | Instead of |
|-----|------------|
| `assertSuccessful()` | `assertStatus(200)` |
| `assertNotFound()` | `assertStatus(404)` |
| `assertForbidden()` | `assertStatus(403)` |
| `assertModelExists()` | `assertDatabaseHas()` |
| `LazilyRefreshDatabase` | `RefreshDatabase` |

### Mocking & Fakes

```php
use function Pest\Laravel\mock;  // always import first

// Always set up fakes AFTER factory setup, not before
$user = User::factory()->create();
Event::fake();
Queue::fake();
Http::fake(['api.example.com/*' => Http::response([...])]);
Http::preventStrayRequests();
```

### Datasets

```php
it('validates email', function (string $email) {
    expect($email)->not->toBeEmpty();
})->with([
    'james' => 'james@example.com',
    'taylor' => 'taylor@example.com',
]);
```

### Model Factories in Tests

- Always use factories, never manually create models without approval
- Check factory custom states before manually setting attributes
- Use `recycle()` to share relationship instances across factories

### Architecture Testing

```php
arch('controllers')
    ->expect('App\Http\Controllers')
    ->toExtendNothing()
    ->toHaveSuffix('Controller');
```

---

## Part 3 — Tailwind CSS v4 (Blade Templates)

### Core Rule

Check existing Tailwind conventions in the project before introducing new patterns.

### v4 Import Syntax

```css
/* v4 */
@import "tailwindcss";

/* NOT v3 directives */
/* @tailwind base; @tailwind components; @tailwind utilities; */
```

### v4 Configuration — CSS-first

```css
@theme {
  --color-brand: oklch(0.72 0.11 178);
}
```
No `tailwind.config.js` needed in v4.

### Replaced Utilities (v3 → v4)

| Deprecated | Replacement |
|------------|-------------|
| `bg-opacity-*` | `bg-black/*` |
| `flex-shrink-*` | `shrink-*` |
| `flex-grow-*` | `grow-*` |
| `overflow-ellipsis` | `text-ellipsis` |

### Spacing

Use `gap` utilities, not margins, for spacing between siblings.

### Dark Mode

If existing pages use dark mode, new pages must too:
```html
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-white">
```

---

## Mandatory Rules

1. Run `vendor/bin/pint --dirty --format agent` after every PHP file change
2. Use `php artisan make:` for all new files — never create PHP classes manually
3. Never delete tests without user approval
4. Always use `$request->validated()` — never `$request->all()`
5. Follow existing project conventions over these defaults when they conflict

## Level History

- **Lv.1** — Base: Combined Laravel best practices, Pest v4 testing, and Tailwind v4. (Origin: 2026-04-21)
- **Lv.2** — Full upgrade: Absorbed all 20 rule files from `laravel-best-practices` skill. Added: Advanced Queries, Caching, Eloquent Patterns, HTTP Client, Collections, Blade & Views, Architecture, Task Scheduling, Migrations, Style/Conventions, Client-Side JS Integration, Pagination Strategy. Removed broken `search-docs` MCP mandatory rule. Archived `laravel-best-practices`. (2026-04-28)
