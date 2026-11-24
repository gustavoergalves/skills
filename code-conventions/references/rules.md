# Code Conventions — Full Rule Catalog

Rules for writing PHP/Laravel code. They cover the semantic content of the code: algorithms,
patterns, naming, composition. The goal is to lower the long-term cost of developing and
maintaining the code:

- more readable code (different parts of the codebase look alike),
- faster code reviews (a reviewer cites a rule number instead of writing an explanation),
- faster decisions while coding (when there are two options, a rule picks one),
- fewer bugs over time.

**Modality.** A rule worded with *must*, *forbidden*, *cannot* or *is not allowed* is
**MUST**: a violation blocks the change. A rule worded with *should*, *avoid* or *if possible*
is **SHOULD**: a violation is reported, but a justified exception is acceptable. Cite rules by
number (e.g. `5.2.3`).

---

## 1. Principles, paradigms and standards

**1.1.** The code must follow the **PSR-1** and **PSR-12** style guides.
<https://www.php-fig.org/psr/psr-1/> · <https://www.php-fig.org/psr/psr-12/>

**1.2.** The code should adhere to the **SOLID**, **KISS** and **DRY** principles.

- **1.2.1. Single Responsibility (SRP)** — each class has only one responsibility.
- **1.2.2. Open-Closed (OCP)** — classes are open for extension, closed for modification.
- **1.2.3. Liskov Substitution (LSP)** — subtypes must be substitutable for their base types.
- **1.2.4. Interface Segregation (ISP)** — clients must not be forced to depend on interfaces
  they do not use.
- **1.2.5. Dependency Inversion (DIP)** — high-level modules do not depend on low-level
  modules; both depend on abstractions. Inject dependencies (constructor injection) instead of
  instantiating them inside the class.
- **KISS** — the simplest solution that solves the problem.
- **DRY** — one piece of knowledge has one place in the code.

**1.3.** The code style must be formatted according to the project's **PHP CodeSniffer** rules.

**1.4.** Use **namespaces** (to avoid naming conflicts) and **autoloading** (to load classes
when needed). No manual `require`/`include` of classes.

**1.5.** Prefer **PSR-compliant libraries**, for compatibility and consistency.

**1.6.** **Parameter, property and return types are a MUST.** Every parameter, every class
property and every return value has a declared type.

---

## 2. Common

**2.1.** **Readability** of the code is more important than "write-ability". Avoid syntactic
sugar if it hurts readability.

**2.2.** The code must be **clear**: avoid magic methods (`__get`, `__call`, `__set`, …) when
possible.

**2.3.** The code must be **clean**. Unused code and commented-out code are forbidden.

**2.4.** The code should be **automatically refactorable** in the IDE: model class annotations
(`@property`, `@property-read`, relation annotations) must be properly declared; variables
resolved by the service container must be properly annotated (`/** @var Foo $foo */`), etc.

**2.5.** The code should have minimal **cyclomatic complexity** (few branches, early returns,
small methods).

**2.6.** The code should be read **from top to bottom**. The reader must not have to keep
something in mind, go back, and reinterpret the code. E.g. `do {} while ()` loops should be
avoided.

**2.7.** Parts of the code (class names, variable names, constant names) must not be **stored
in the database**, because that makes automatic refactoring impossible. Exception: Eloquent
polymorphic relations.

**2.8.** Do not use `exit`, `die`, or any other operator that terminates or alters the running
process.

**2.9.** Any script/command that **changes data** must ask for **confirmation** before acting
and must **log** the result of the work.

```php
// Bad
$repository->deleteItems();

// Good
if (!$this->confirm('Delete?')) {
    $this->info('Delete canceled, exit');
    return;
}
$deletedCount = $repository->deleteItems();
Log::info("{$deletedCount} items deleted.");
```

---

## 3. Layered architecture

**3.1.** The code must follow a layered architecture. Each layer has one responsibility:

| Layer | Responsibility | Lives here |
| --- | --- | --- |
| **Presentation** | User input and output: web views, APIs, CLI | Views, Request / Response objects (FormRequests, API Resources) |
| **Application** | Application flow, workflows | Routing, bootstrapping, Controllers, Commands |
| **Domain** | Domain entities and their behavior, business rules | Services, Models, domain interfaces, domain exceptions |
| **Infrastructure** | Implementation details: database, external services, file system | Eloquent queries, Repositories, API clients |

**3.2.** The **Domain layer must not depend on any other layer**. It declares interfaces
(still part of the Domain layer) that other layers implement — e.g. the Domain declares
`UserRepositoryInterface`, Infrastructure implements it as `UserRepository`.

**3.3.** **Controllers and Commands must be as thin as possible** and must not contain business
logic: read the request/arguments, delegate to the Domain, return the response.

Reference example (applies 1.2.5, 1.6, 3.1–3.3, 10.3, 11.4):

```php
// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Http\Requests\UserIndexRequest;
use App\Services\UserService;
use Illuminate\Http\JsonResponse;

class UserController extends Controller
{
    public function __construct(private readonly UserService $userService)
    {
    }

    public function index(UserIndexRequest $request): JsonResponse
    {
        $usernameFilter = $request->validated('usernameFilter');

        $users = $this->userService->loadUsers($usernameFilter);

        return response()->json(['users' => $users]);
    }
}
```

```php
// app/Services/UserService.php
namespace App\Services;

use App\Models\User;
use App\Repositories\UserRepositoryInterface;
use Illuminate\Support\Collection;

class UserService
{
    public function __construct(private readonly UserRepositoryInterface $userRepository)
    {
    }

    /**
     * @return Collection<int, User>
     */
    public function loadUsers(?string $usernameFilter = null): Collection
    {
        return $this->userRepository->loadAll($usernameFilter);
    }
}
```

```php
// app/Repositories/UserRepositoryInterface.php
namespace App\Repositories;

use App\Models\User;
use Illuminate\Support\Collection;

interface UserRepositoryInterface
{
    /**
     * @return Collection<int, User>
     */
    public function loadAll(?string $usernameFilter = null): Collection;
}
```

```php
// app/Repositories/UserRepository.php
namespace App\Repositories;

use App\Models\User;
use Illuminate\Support\Collection;

class UserRepository implements UserRepositoryInterface
{
    /**
     * @return Collection<int, User>
     */
    public function loadAll(?string $usernameFilter = null): Collection
    {
        $query = User::query();
        if ($usernameFilter !== null) {
            $query->where('username', 'ILIKE', "{$usernameFilter}%");
        }

        return $query->get();
    }
}
```

---

## 4. Pull requests

**4.1.** A pull request should contain **as few lines of code as possible**.

**4.2.** A pull request should not contain code **unrelated to the task** (enforced "if
possible" today; it will become stricter). No drive-by refactors, reformatting or renames of
untouched code.

**4.3.** A pull request **must not change more than 12 files**. If the work needs more:
- the task probably needs to be **split into smaller tasks**, or
- the change must be **split into several PRs** (e.g. contract/interface first, then
  implementation, then wiring; or one PR per independent capability), each one reviewable and
  mergeable on its own.

Every changed file counts (source, tests, migrations, config).

---

## 5. Variables

**5.1.** Variables must be written in **camelCase**.

### 5.2. Naming

**5.2.1.** Names must be **informative**. Short names like `$t` are not allowed. The name must
match the content.

```php
// Bad
$total = ['sold' => 23, 'canceled' => 11, 'pr' => 450];
$et = 5;

// Good
$dailySaleStatistic = ['sold' => 23, 'canceled' => 11, 'processed' => 450];
$expectedTime = 5;
```

**5.2.2.** Names should be in **correct English** when possible.

```php
$usersStored = [];   // Bad
$storedUsers = [];   // Good
```

**5.2.3.** A variable name must **not contain its type**.

```php
// Bad
$projectsCollection = $repository->loadProjects();
$projectsArray = $repository->loadProjects();
$projectIdsArray = $utils->extractField('id', $projectsList);
$messageString = '';

// Good
$projects = $repository->loadProjects();
$projectsIds = $utils->extractField('id', $projects);
$message = '';
```

**5.2.4.** Variables for the same entity must have **the same name across the codebase**.

```php
// Bad
$orderProduct = new UserOrderProduct();
$lineItem = new UserOrderProduct();
$userOrderProduct = new UserOrderProduct();

// Good — one name everywhere
$orderProduct = new UserOrderProduct();
```

**5.2.5.** Variables holding a property of an object must **include the object's name**.

```php
// Bad
$name = $project->name;
$type = $project->type;

// Good
$projectName = $project->name;
$projectType = $project->type;
```

**5.2.6.** *(Revoked — struck out in the source document; do not enforce, do not report.)*
"Object variables and properties must be nouns", e.g. `expirationDate` instead of `expireAt`,
`setExpirationDate()` instead of `setExpireAt()`.

**5.3.** A **scalar** variable must not be passed **by reference** (`&`).

```php
// Bad
function removePrefix(string &$name): void
{
    $name = str_replace('prefix', '', $name);
}

// Good
function removePrefix(string $name): string
{
    return str_replace('prefix', '', $name);
}
```

**5.4.** **Method parameters must not be reassigned.** Use a new variable.

```php
// Bad
function parseText(string $text): void
{
    $text = trim($text);
}

// Good
function parseText(string $text): void
{
    $trimmedText = trim($text);
}
```

**5.5.** **One variable declaration per line.**

```php
$foo = false; $bar = true;   // Bad

$foo = false;                // Good
$bar = true;
```

**5.6.** Using the **result of an assignment** is forbidden (chained assignment, assignment
inside an argument or condition).

```php
// Bad
$foo = $bar = strlen($someVar);
$this->callSomeFunc($bar = strlen($foo));

// Good
$bar = strlen($someVar);
$foo = $bar;

$bar = strlen($foo);
$this->callSomeFunc($bar);
```

**5.7.** The purpose of every **numeric literal** (magic number) must be clear from context —
name it with a variable or constant.

```php
// Bad
$isOnlyDeleted = 1;
if ($object->isDeleted == $isOnlyDeleted) { }
for ($i = 0; $i < 5; $i++) { }

// Good
if ($object->isDeleted) { }
$apiMaxRetryLimit = 5;
for ($i = 0; $i < $apiMaxRetryLimit; $i++) { }
```

**5.8.** Use **`null` for a missing scalar value**. `0` and `''` must not mean "missing".

```php
// Bad
function sendMessage(string $body, string $header = '', string $footer = '', int $delay = 0): void

// Good
function sendMessage(string $body, ?string $header = null, ?string $footer = null, ?int $delay = null): void
```

---

## 6. Boolean variables and methods

**6.1.** Boolean variables and methods must start with a verb: **`is`, `has` or `can`**.

```php
$userValid = $user->valid();         // Bad
$isUserValid = $user->isValid();     // Good
```

**6.2.** **Negative boolean names are forbidden.**

```php
// Bad
if ($project->isInvalid()) { }
if ($project->isNotValid()) { }
if ($accessManager->isAccessDenied()) { }

// Good
if (!$project->isValid()) { }
if (!$accessManager->isAccessAllowed()) { }
if ($accessManager->canAccess()) { }
```

**6.3.** Avoid **boolean flags as function parameters** when possible — move the decision to the
caller.

```php
// Bad
storeUser($user, $isProjectNotificationEnabled);

function storeUser(User $user, bool $isNotificationEnabled): void
{
    // ...
    if ($isNotificationEnabled) {
        notify('new user');
    }
}

// Good
storeUser($user);
if ($notificationManager->isProjectNotificationEnabled($project)) {
    notify('new user');
}

function storeUser(User $user): void
{
    // ...
}
```

---

## 7. Arrays

**7.1.** The **`+` operator must not be used to merge arrays.** Use `array_merge`.

```php
return $initialData + $loadedData;              // Bad
return array_merge($initialData, $loadedData);  // Good
```

**7.2.** To check whether a key exists in an associative array use **`array_key_exists`**, not
`isset`.

```php
if (isset($requestData['project_key'])) { }                // Bad
if (array_key_exists('project_key', $requestData)) { }     // Good
```

---

## 8. Strings

**8.1.** Use **single quotes**. Double quotes only when the string has:
- interpolation, or
- special characters `\n`, `\r`, `\t`,

(and, as the source example shows, to avoid escaping single quotes inside the string).

```php
// Bad
$string = "Some string";
$string = 'Some \'string\'';
$string = "\t" . 'Some string' . "\n";

// Good
$string = 'Some string';
$string = "Some 'string'";
$string = "\tSome string\n";
```

---

## 9. Dates

**9.1.** A date is always a **`DateTimeImmutable`**; an interval is an **immutable interval**
(`DateIntervalImmutable` in the source; PHP has no native class of that name, so use the
project's immutable interval type if one exists — e.g. `CarbonInterval` treated as immutable —
otherwise a `DateInterval` built once and never mutated). Never timestamps or seconds as ints.

```php
// Bad
$now = time();
$interval = 86400 * 30;

// Good
$now = new DateTimeImmutable();
$interval = new DateInterval('P30D');
```

**9.2.** If a date must be stored as a scalar, use a **string**, not an int timestamp.

```php
$user->creation_date = time();                   // Bad
$user->creation_date = '2018-01-18 12:54:11';    // Good
```

**9.3.** Intervals must **not use months or years** (their length varies). Use days.

```php
new DateInterval('P2M');    // Bad
new DateInterval('P60D');   // Good
```

---

## 10. Methods

**10.1.** Use the **maximum possible type hinting**:
- all parameter types declared in the signature,
- return type declared in the signature,
- thrown exceptions declared in PHPDoc (`@throws`),
- array key/value types declared in PHPDoc (`array<string>`, `array<int, User>`, …).

```php
// Bad
function makeCoffee($type, $volume, $additions)

// Good
/**
 * @param array<string> $additions
 * @throws OutOfSugarException
 */
function makeCoffee(string $type, int $volume, array $additions): Coffee
```

**10.2.** A method name **starts with a verb** and follows the variable-naming rules (exception:
Eloquent model relations, e.g. `items()`).

```php
public function items()                                  // Bad (not a relation)
public function convertedDataObject(array $data)         // Bad

public function loadItems()                              // Good
public function convertDataToObject(array $data)         // Good
```

**10.3.** All methods are **`private` by default**. `protected` only if a child class uses it;
`public` only if other classes use it.

**10.4.** **Recursion** only in exceptional cases.

**10.5.** Parameter order: **required → frequently used → rarely used**.

**10.6.** Nullable parameters are marked with **`?`**, even when the default is `null`.

```php
function calcExample(int $number = null) { }     // Bad
function calcExample(?int $number = null) { }    // Good
```

**10.7.** A method returns **one data structure, or `null`, or nothing (`void`)**. Mixed return
types and arrays of mixed/unclear values are forbidden.

```php
// Bad
function loadUser(): mixed
{
    if ($someCondition) {
        return ['id' => 1];
    }
    return new User();
}

// Good
function loadUser(): User
{
    if ($someCondition) {
        $user = new User();
        $user->id = 1;
        return $user;
    }
    return new User();
}
```

**10.8.** A method returning a single object or scalar returns **`null` when not found** (not
`false`, `0`, `''`, `[]`).

```php
// Bad
function getUser(int $userId): mixed
{
    if ($someCondition) {
        return false;
    }
    return $this->userRepository->getById($userId);
}

// Good
function getUser(int $userId): ?User
{
    if ($someCondition) {
        return null;
    }
    return $this->userRepository->getById($userId);
}
```

**10.9.** In a large method (**more than 15 lines**), if the return value is modified along the
way, name it **`$result`**, so it is always clear where you work with the result and where with
local variables.

```php
// Bad
function loadUsers(): array
{
    $users = [];
    // ... a lot of code modifying $users
    return $users;
}

// Good
function loadUsers(): array
{
    $users = $this->getUsers();
    // ...
    $result = $this->filterUsers($users);
    // ... a lot of code modifying $result
    return $result;
}
```

**10.10.** A method must clearly **distinguish normal situations from exceptional ones**: an
error throws; an empty-but-valid result returns empty.

```php
// Bad
if ($connectionError !== null) {
    return [];              // error is lost
}

// Good
if ($connectionError !== null) {
    throw new ConnectionError();
}
if (count($data) === 0) {
    return [];              // legitimately empty
}
```

**10.11.** Method structure: **check parameters → receive data → work → result**.

**10.12.** A method with many optional parameters must not be replaced with a **chain object**
(fluent builder) — use a **DTO** instead.

---

## 11. Classes

**11.1.** Traits have the **`Trait` suffix** (`HasUuidTrait`).

**11.2.** Interfaces have the **`Interface` suffix** (`UserRepositoryInterface`).

**11.3.** Abstract classes have the **`Abstract` prefix** (`AbstractImporter`).

**11.4.** All properties and constants are **`private` by default**. `protected` only if a
child class uses it; `public` only if other classes use it.

**11.5.** Members are ordered **by visibility, top to bottom: `public` → `protected` →
`private`** (properties first, then methods, each group in that order).

```php
// Good
class SomeClass
{
    public $pubPropA;
    protected $protPropA;
    private $privPropA;

    public function pubA() { }
    public function pubB() { }
    protected function protA() { }
    private function privA() { }
}
```

**11.6.** **Static access only through the class name.** An instance accesses only its own
properties and methods.

```php
$type = $user::TYPE;   // Bad
$type = User::TYPE;    // Good
```

**11.7.** Avoid **aliases in `use` statements** (`use Foo\Bar as Baz`) whenever possible.

---

## 12. Comments

**12.1.** An unavoidable **hack** is marked with an explanatory comment tagged **`// hack:`**.

```php
// hack: the carrier API returns 200 with an error body on rate limit; retry on that body
```

**12.2.** An algorithm **taken from an external source** is marked with a **link to the
source**.

**12.3.** **No unnecessary comments.** Code must be easy to read, understand, and
self-explanatory, so it does not need comments explaining what it does. This applies to the
names of classes, methods, variables, constants, etc.: if a comment is needed to explain what
something does or holds, rename it or extract a well-named method/variable/constant instead.

Forbidden:
- comments that restate the code (`// get the user`, `// loop through orders`,
  `// return result`);
- comments that explain a bad name (`$d = 30; // days until expiration`);
- section/step banners inside a method (`// Step 1: validate`) — extract methods instead;
- PHPDoc that only repeats the signature (`@param string $name The name`,
  `@return void`) or describes the method name in prose;
- commented-out code (2.3), changelog/author/"added by"/"fixed bug X" comments.

Allowed (the only cases):
- `// hack:` comments (12.1) and source links (12.2);
- PHPDoc that carries **type information the language cannot express**: array shapes and
  generics, `@throws`, `@property` model annotations, `@var` for container-resolved
  variables (10.1, 2.4);
- a short **why** comment for a non-obvious constraint that cannot be expressed in code
  (external API quirk, legal/business rule with no other home). Never a *what* comment.

```php
// Bad
// Check if the user can receive the discount
if ($user->orders_count > 10 && $user->created_at < now()->subDays(365)) { }

// Good
if ($this->isEligibleForLoyaltyDiscount($user)) { }
```

---

## 13. Exceptions

**13.1.** **Third-party library exceptions are caught immediately** at the call site and
wrapped in a project exception (keeping the original as `previous`).

```php
// Bad
$objects = $facebookAds->requestData($params);

// Good
try {
    $objects = $facebookAds->requestData($params);
} catch (FacebookAdsException $e) {
    throw new ExternalServiceError('Facebook error', 0, $e);
}
```

**13.2.** By default, exception data/messages are **not shown to the user**.

**13.3.** Each **domain area has its own exception classes**.

**13.4.** The exception **message does not contain context data** (ids, values). Pass context
through a specific context parameter/property instead.

```php
// Bad
throw new OrderNotFoundException("Order {$orderId} not found");

// Good — the domain exception exposes a context parameter
throw new OrderNotFoundException('Order not found', ['orderId' => $orderId]);
```

---

## 14. Conditions

**14.1.** The condition of an `if` must be **boolean only**.

```php
// Bad
if (count($userProjects)) { }
if ($project) { }

// Good
if ($isResponseError) { }
if ($response->isError()) { }
if (count($userProjects) > 0) { }
```

**14.2.** Compare non-boolean values with **strict comparison (`===`, `!==`)** and explicit
casting. No loose comparison, no implicit casting.

```php
// Bad
if ($project) { }
if ($request->postData('sum') == 100) { }
if (!$request->postData('sum')) { }
if (!$bill->comment) { }

// Good
if ($project === null) { }
if ((int) $request->postData('sum') === 100) { }
if ($bill->comment === '') { }
```

**14.3.** Do **not compare booleans with `true`/`false`**.

```php
if ($bill->isPaid() == true) { }   // Bad
if ($bill->isPaid()) { }           // Good
```

**14.4.** Check for the **presence of a positive case**, not the absence of negative ones.

```php
if (!is_numeric($value) && !is_object($value)) { }     // Bad
if (is_string($value) && $value !== '') { }            // Good
```

**14.5.** When mixing **AND and OR**, make precedence explicit with **parentheses**.

```php
if ($isMobile || $isSizeTooBig && $isAllowedToShrink) { }       // Bad
if (($isMobile || $isSizeTooBig) && $isAllowedToShrink) { }     // Good
if ($isMobile || ($isSizeTooBig && $isAllowedToShrink)) { }     // Good
```

**14.6.** Avoid **chains of ternary operators**.

---

## 15. Tests

**15.1.** New or changed behavior is covered by tests:
- **Unit tests** verify the behavior of individual functions or classes in isolation.
- **Integration tests** verify the interaction between parts of the application: database,
  external services, APIs.
