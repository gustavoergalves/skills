# GWT Examples

## Multi-step Given (reaching a starting state via prior calls)

```php
public function testRingingTheBellOnAnExistingFavouriteUpdatesItInPlace(): void
{
    $lot = $this->givenEligibleLot();
    $this->givenLotIsFavourited($lot);

    $this->whenBellIsRungFor($lot);

    $this->thenFavouriteWasUpdatedNotCreated();
    $this->thenNotificationsAreEnabled();
}

private function givenEligibleLot(): AuctionObject
{
    return AuctionObject::factory()->eligible()->create();
}

private function givenLotIsFavourited(AuctionObject $lot): void
{
    $this->service->addForCustomer(self::LINKED_SUB, 99, $lot->rambase_id, $this->tenantId('skanfil'));
}

private function whenBellIsRungFor(AuctionObject $lot): void
{
    $this->result = $this->service->setNotifyForCustomer(
        self::LINKED_SUB,
        99,
        $lot->rambase_id,
        $this->tenantId('skanfil'),
        notify: true,
    );
}

private function thenFavouriteWasUpdatedNotCreated(): void
{
    $this->assertFalse($this->result['created']);
    $this->assertDatabaseCount('watchlist_items', 1);
}

private function thenNotificationsAreEnabled(): void
{
    $this->assertDatabaseHas('watchlist_items', ['notify_enabled' => true]);
}
```

`givenLotIsFavourited` calls the same service as the action under test, but it is not what
this test verifies, so it is a Given step.

## Unit test with a collaborator spy

Unit tests (not feature tests) may replace a collaborator. Prefer a spy so the setup lives in
Given and the verification lives in Then, instead of hiding the outcome in a mock
expectation registered up front.

```php
public function testSyncDispatchesOneJobPerPage(): void
{
    $this->givenRamBaseReturnsPages($this->pageOne(), $this->pageTwo());

    $this->whenSyncRuns();

    $this->thenSyncPageJobsAreDispatched(2);
}

private function givenRamBaseReturnsPages(array ...$pages): void
{
    $this->client = $this->createMock(RamBaseClient::class);
    $this->client->method('fetchPage')->willReturnOnConsecutiveCalls(...$pages);
}

private function whenSyncRuns(): void
{
    (new SyncService($this->client))->run();
}

private function thenSyncPageJobsAreDispatched(int $count): void
{
    Queue::assertPushed(SyncPageJob::class, $count);
}
```

## HTTP feature test

```php
public function testCreatingASavedSearchReturns201(): void
{
    $user = $this->givenUser();

    $this->whenSavedSearchIsCreatedBy($user, name: 'My search');

    $this->thenSavedSearchIsStored('My search');
}

private function givenUser(): User
{
    return User::factory()->create();
}

private function whenSavedSearchIsCreatedBy(User $user, string $name, array $criteria = []): void
{
    $this->response = $this->actingAs($user)
        ->postJson('/api/v1/saved-searches', ['name' => $name, 'criteria' => $criteria]);
}

private function thenSavedSearchIsStored(string $name): void
{
    $this->response->assertStatus(Response::HTTP_CREATED);
    $this->assertDatabaseHas('saved_searches', ['name' => $name]);
}
```

## Data provider

The provider is the parameterised Given; its values flow straight into the steps.

```php
/**
 * @dataProvider invalidCriteriaProvider
 */
public function testInvalidCriteriaIsRejected(array $criteria, string $expectedError): void
{
    $user = $this->givenUser();

    $this->whenSavedSearchIsCreatedBy($user, name: 'x', criteria: $criteria);

    $this->thenValidationFailsOn($expectedError);
}

public static function invalidCriteriaProvider(): array
{
    return [
        'missing category' => [['brand' => 'skanfil'], 'criteria.category'],
        'unknown brand' => [['category' => 'x', 'brand' => 'nope'], 'criteria.brand'],
    ];
}

private function thenValidationFailsOn(string $field): void
{
    $this->response->assertStatus(Response::HTTP_UNPROCESSABLE_ENTITY);
    $this->response->assertJsonValidationErrors($field);
}
```

`givenUser` and `whenSavedSearchIsCreatedBy` are the same steps as the HTTP example above —
reuse them within the class rather than writing variants.

## Class layout

Tests first, then steps grouped Given → When → Then. The When result lives in a private
property.

```php
class SavedSearchControllerTest extends TestCase
{
    use RefreshDatabase;

    private TestResponse $response;

    public function testCreatingASavedSearchReturns201(): void { /* ... */ }

    public function testInvalidCriteriaIsRejected(array $criteria, string $expectedError): void { /* ... */ }

    public static function invalidCriteriaProvider(): array { /* ... */ }

    private function givenUser(): User { /* ... */ }

    private function whenSavedSearchIsCreatedBy(User $user, string $name, array $criteria = []): void { /* ... */ }

    private function thenSavedSearchIsStored(string $name): void { /* ... */ }

    private function thenValidationFailsOn(string $field): void { /* ... */ }
}
```
