---
name: eloquent-models
description: Use when adding or refactoring query logic on Eloquent models
---
# Rules
- **Never** use local query scopes on models. This means:
    - No `public function scopeXxx(Builder $query, ...)` methods.
    - No `#[Scope]` PHP attribute (Laravel 11+).
- Whenever a scope-like reusable query constraint is needed, add it to a model-specific `QueryBuilder` class instead.
- One builder per model. Do not share a builder across multiple models.

# Where it lives
- Builder class path: `app/Models/Builders/{ModelName}QueryBuilder.php`
- Namespace: `App\Models\Builders`
- Class name: `{ModelName}QueryBuilder` (e.g. `UserShopifyStoreQueryBuilder` for `UserShopifyStore`).

# Method naming
- Constraint methods are prefixed with `where` (e.g. `whereStatusActive()`, `whereStoreType(StoreType $type)`).
- Eager-load / join helpers are prefixed with `with` (e.g. `withActiveOwner()`).
- Every method must return `self` so calls remain chainable.

# Wiring it up
On the model, override `newEloquentBuilder` to return the custom builder. Add a `@method` PHPDoc on the model so IDE autocomplete and PHPStan resolve the builder methods on `Model::query()`.

```php
use App\Builders\UserShopifyStoreQueryBuilder;
use Illuminate\Database\Eloquent\Model;

/**
 * @method static UserShopifyStoreQueryBuilder query()
 */
class UserShopifyStore extends Model implements Authenticatable
{
    /**
     * @param  \Illuminate\Database\Query\Builder  $query
     */
    public function newEloquentBuilder($query): UserShopifyStoreQueryBuilder
    {
        return new UserShopifyStoreQueryBuilder($query);
    }
}
```

```php
namespace App\Builders;

use App\Models\UserShopifyStore;
use Illuminate\Database\Eloquent\Builder;

/**
 * @template TModelClass of UserShopifyStore
 * @extends Builder<TModelClass>
 */
class UserShopifyStoreQueryBuilder extends Builder
{
    public function whereStoreType(StoreType $storeType): self
    {
        return $this->where(['store_type' => $storeType->value]);
    }

    public function whereStatusActive(): self
    {
        return $this->whereStatus(StoreStatus::Enabled);
    }

    public function whereStatus(StoreStatus $storeStatus): self
    {
        return $this->where(['status' => $storeStatus->value]);
    }
}
```

# Anti-pattern (do NOT do this)

```php
// ❌ Local scope method on the model
public function scopeActive(Builder $query): Builder
{
    return $query->where('status', StoreStatus::Enabled->value);
}

// ❌ Scope attribute on the model
#[Scope]
protected function active(Builder $query): void
{
    $query->where('status', StoreStatus::Enabled->value);
}
```