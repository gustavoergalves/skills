# GWT Examples

## Multi-step Given (reaching a starting state via prior calls)

```php
public function test_ringing_the_bell_on_an_existing_favourite_updates_it_in_place(): void
{
    // Given
    $lot = $this->eligibleLot();
    $this->service->addForCustomer(self::LINKED_SUB, 99, $lot->rambase_id, $this->tenantId('skanfil'));

    // When
    $result = $this->service->setNotifyForCustomer(self::LINKED_SUB, 99, $lot->rambase_id, $this->tenantId('skanfil'), true);

    // Then
    $this->assertFalse($result['created']);
    $this->assertDatabaseCount('watchlist_items', 1);
    $this->assertDatabaseHas('watchlist_items', ['notify_enabled' => true]);
}
```

The setup call (`addForCustomer`) uses the same service as the call under test
(`setNotifyForCustomer`) but is not what this test verifies, so it stays under **Given**.

## Repository/unit test with a mock collaborator

Unit tests (not feature tests) may mock a collaborator. The mock expectation is part of the
precondition, so it goes under **Given**; only the call being verified goes under **When**.

```php
public function test_sync_dispatches_one_job_per_page(): void
{
    // Given
    $client = $this->createMock(RamBaseClient::class);
    $client->expects($this->exactly(2))
        ->method('fetchPage')
        ->willReturnOnConsecutiveCalls($this->pageOne(), $this->pageTwo());
    $service = new SyncService($client);

    // When
    $service->run();

    // Then
    Queue::assertPushed(SyncPageJob::class, 2);
}
```

## HTTP feature test

```php
public function test_creating_a_saved_search_returns_201(): void
{
    // Given
    $user = User::factory()->create();

    // When
    $response = $this->actingAs($user)
        ->postJson('/api/v1/saved-searches', ['name' => 'My search', 'criteria' => []]);

    // Then
    $response->assertStatus(Response::HTTP_CREATED);
    $this->assertDatabaseHas('saved_searches', ['name' => 'My search']);
}
```

## Data provider

```php
/**
 * @dataProvider invalidCriteriaProvider
 */
public function test_invalid_criteria_is_rejected(array $criteria, string $expectedError): void
{
    // Given
    $user = User::factory()->create();

    // When
    $response = $this->actingAs($user)
        ->postJson('/api/v1/saved-searches', ['name' => 'x', 'criteria' => $criteria]);

    // Then
    $response->assertStatus(Response::HTTP_UNPROCESSABLE_ENTITY);
    $response->assertJsonValidationErrors($expectedError);
}

public static function invalidCriteriaProvider(): array
{
    return [
        'missing category' => [['brand' => 'skanfil'], 'criteria.category'],
        'unknown brand' => [['category' => 'x', 'brand' => 'nope'], 'criteria.brand'],
    ];
}
```
