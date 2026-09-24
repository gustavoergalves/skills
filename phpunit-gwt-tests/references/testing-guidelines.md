# Testing Guidelines

General conventions for Laravel/PHPUnit test suites, with every example in Given/When/Then
shape. If anything here conflicts with the repo's `CLAUDE.md` (e.g. test method naming),
`CLAUDE.md` wins.

## Test types and namespaces

| Type | Namespace | Scope |
|---|---|---|
| Unit | `Tests\Unit\...` | One component in isolation |
| Feature | `Tests\Feature\...` | Components working together inside the app |
| Integration | `Tests\Integration\...` | The **only** tests allowed to call external APIs; excluded from normal test runs |

The namespace after the type prefix mirrors the class under test:

- `App\Services\UserService` → `Tests\Unit\Services\UserServiceTest` or `Tests\Feature\Services\UserServiceTest`
- `App\Http\Controllers\UserController` → `Tests\Feature\Http\Controllers\UserControllerTest`

```php
namespace Tests\Feature\Http\Controllers;

use Illuminate\Foundation\Testing\WithFaker;
use Tests\TestCase;

class UserControllerTest extends TestCase
{
    use WithFaker;

    public function test_stores_user_data_correctly(): void
    {
        // Given
        $data = [
            'name' => $this->faker->name,
            'email' => $this->faker->safeEmail,
        ];

        // When
        $response = $this->post('/user', $data);

        // Then
        $response->assertRedirect('/home');
        $this->assertDatabaseHas('users', [
            'email' => $data['email'],
            'name' => $data['name'],
        ]);
    }
}
```

## Mandatory feature tests

Every one of these needs feature tests before it ships:

- Controllers
- Jobs
- EventListeners
- Commands
- ApiClients
- Services

## Database isolation

Tests run inside a transaction that is rolled back after each test, so one test cannot
affect another — no manual DB cleanup under **Then**.

## TestData classes (fixture data)

Put sample data (API payloads, raw responses, data variants) in `Tests\TestData\...`
classes with static methods, not inline in the test. This keeps test logic separate from
test data and lets tests reuse the same variants. Loading it belongs under **Given**.

```php
namespace Tests\TestData\Shopify\Inventory;

class InventoryLevelTestData
{
    /**
     * @return array<string, mixed>
     */
    public static function getInventoryLevelsResponse(): array
    {
        $response = <<<JSON
        {
          "inventory_levels": [
            {
              "inventory_item_id": 49148385,
              "location_id": 655441491,
              "available": 2,
              "updated_at": "2023-07-05T18:38:03-04:00",
              "admin_graphql_api_id": "gid://shopify/InventoryLevel/655441491?inventory_item_id=49148385"
            }
          ]
        }
        JSON;

        return json_decode($response, true);
    }
}
```

```php
public function test_get_inventory_levels(): void
{
    // Given
    $inventoryLevelResponse = InventoryLevelTestData::getInventoryLevelsResponse();
    Http::fake([
        '*.myshopify.com/admin/api/*/inventory_levels.json*' => Http::response($inventoryLevelResponse),
    ]);

    // When
    $inventoryLevels = $this->inventoryClient->getInventoryLevels(87324);

    // Then
    $this->assertIsArray($inventoryLevels);
    $this->assertEquals(
        $inventoryLevelResponse['inventory_levels'][0]['inventory_item_id'],
        $inventoryLevels[0]['inventory_item_id'],
    );
}
```

## Assertions

Assert thoroughly: check everything that can go wrong (or right), not just the happy-path
return value — status, persisted state, dispatched jobs/events, side effects.

## Mocking with Mockery

Use Mockery to stand in for databases or external services so the test focuses on the code
under test. Mocks must reflect the real dependency's behaviour, and **must verify incoming
parameters** (`withArgs` / `with`) and call counts (`times`), not just stub return values.
The mock setup is a precondition, so it goes under **Given**; the `times()` expectations are
verified automatically at teardown.

```php
public function test_purchase_sends_one_paid_plan_and_two_recurring_events(): void
{
    // Given
    $this->mock(ImpactApiClient::class, function (MockInterface $mock) {
        $mock->shouldReceive('sendTrackingConversionEvent')
            ->withArgs(fn ($event) => $event instanceof PaidPlanPurchaseEvent)
            ->times(1);

        $mock->shouldReceive('sendTrackingConversionEvent')
            ->withArgs(fn ($event) => $event instanceof RecurringSubscriptionEvent)
            ->times(2);
    });

    // When
    $this->service->handlePurchase($this->subscription());

    // Then
    // (expectations above are verified by Mockery on teardown)
}
```

## Final notes

- **Keep it clean:** use `tearDown` to remove created files and other artifacts; the
  environment must be fresh for the next test.
- **Cover the bases:** every component type in the mandatory list above.
- **Detail your tests:** a short comment on *why* a test exists (above the method, not
  inside the GWT sections) saves time later.
