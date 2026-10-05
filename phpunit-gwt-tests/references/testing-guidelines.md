# Testing Guidelines

General conventions for Laravel/PHPUnit test suites, with every example in Given/When/Then
shape (extracted `given*`/`when*`/`then*` steps, no GWT comments). If anything here conflicts with the repo's `CLAUDE.md` (e.g. test method naming),
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
use Illuminate\Testing\TestResponse;
use Tests\TestCase;

class UserControllerTest extends TestCase
{
    use WithFaker;

    private TestResponse $response;

    public function testStoresUserDataCorrectly(): void
    {
        $data = $this->givenUserData();

        $this->whenUserIsSubmitted($data);

        $this->thenUserIsStoredAndRedirectedHome($data);
    }

    private function givenUserData(): array
    {
        return [
            'name' => $this->faker->name,
            'email' => $this->faker->safeEmail,
        ];
    }

    private function whenUserIsSubmitted(array $data): void
    {
        $this->response = $this->post('/user', $data);
    }

    private function thenUserIsStoredAndRedirectedHome(array $data): void
    {
        $this->response->assertRedirect('/home');
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
affect another — no manual DB cleanup in `then*` steps.

## TestData classes (fixture data)

Put sample data (API payloads, raw responses, data variants) in `Tests\TestData\...`
classes with static methods, not inline in the test. This keeps test logic separate from
test data and lets tests reuse the same variants. Loading it belongs in a `given*` step.

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
public function testGetInventoryLevels(): void
{
    $shopifyResponse = $this->givenShopifyReturnsInventoryLevels();

    $this->whenInventoryLevelsAreFetched(locationId: 87324);

    $this->thenInventoryLevelsMatch($shopifyResponse);
}

private function givenShopifyReturnsInventoryLevels(): array
{
    $response = InventoryLevelTestData::getInventoryLevelsResponse();
    Http::fake([
        '*.myshopify.com/admin/api/*/inventory_levels.json*' => Http::response($response),
    ]);

    return $response;
}

private function whenInventoryLevelsAreFetched(int $locationId): void
{
    $this->inventoryLevels = $this->inventoryClient->getInventoryLevels($locationId);
}

private function thenInventoryLevelsMatch(array $shopifyResponse): void
{
    $this->assertIsArray($this->inventoryLevels);
    $this->assertEquals(
        $shopifyResponse['inventory_levels'][0]['inventory_item_id'],
        $this->inventoryLevels[0]['inventory_item_id'],
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
Prefer a spy (`$this->spy(...)`) so stubbing goes in a `given*` step and the
`shouldHaveReceived()->withArgs()->times()` verification goes in a `then*` step — the
expected outcome stays visible in the test body instead of hiding in up-front mock
expectations.

```php
public function testPurchaseSendsOnePaidPlanAndTwoRecurringEvents(): void
{
    $this->givenImpactApiIsSpied();

    $this->whenPurchaseIsHandled();

    $this->thenConversionEventsAreSent(PaidPlanPurchaseEvent::class, times: 1);
    $this->thenConversionEventsAreSent(RecurringSubscriptionEvent::class, times: 2);
}

private function givenImpactApiIsSpied(): void
{
    $this->impactApi = $this->spy(ImpactApiClient::class);
}

private function whenPurchaseIsHandled(): void
{
    $this->service->handlePurchase($this->subscription());
}

private function thenConversionEventsAreSent(string $eventClass, int $times): void
{
    $this->impactApi->shouldHaveReceived('sendTrackingConversionEvent')
        ->withArgs(fn ($event) => $event instanceof $eventClass)
        ->times($times);
}
```

## Final notes

- **Keep it clean:** use `tearDown` to remove created files and other artifacts; the
  environment must be fresh for the next test.
- **Cover the bases:** every component type in the mandatory list above.
- **Detail your tests:** a short comment on *why* a test exists (above the method, not
  inside the test body) saves time later.
