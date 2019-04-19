
# PHP RFC: Algebraic Types

  - Date: 2019-04-19
  - Author: Kirill Nesmeyanov (nesk@xakep.ru)
  - Status: Draft

## Introduction

The PHP already provides a set of types that are intersections of 
existing ones ([pseudo-types](https://www.php.net/manual/en/language.pseudo-types.php)), 
but they do not regulate their hierarchy and the possibility of overload. 
In addition, attempts to implement intersections that have been rejected or not 
completed to varying degrees have been repeatedly made:

  - Draft: [Mixed typehint](https://wiki.php.net/rfc/mixed-typehint)
  - Declined: [Union Types](https://wiki.php.net/rfc/union_types)
  - Declined: [Array Of](https://wiki.php.net/rfc/arrayof)
  - Under Discussion: [Intersection Types](https://wiki.php.net/rfc/intersection_types)
  - Under Discussion: [Scalar Pseudo-type](https://wiki.php.net/rfc/scalar-pseudo-type)
  - Under Discussion: [Resource typehint](https://wiki.php.net/rfc/resource_typehint)

In addition, there are several RFCs that have already been taken, for 
example `object` or `iterable` type hints.

This RFC proposes to regulate the type hierarchy and allow declaring 
them by yourself using `type` keyword. 

For example, take the existing implementation of `iterable` type:

```php
interface UsersRepositoryInterface
{
    public function all(): iterable;
}
```

In this example, we see a reference to iterable, which can be decomposed 
into two different types: `array` and `\Traversable`. In this case, we can 
define this type using the proposed solution:

```php
type iterable = array | \Traversable;
```

## Proposal

Add the ability to describe types using the keyword `type`:

```php
type scalar = bool | int | float | string | null;

class User
{
    private string $name;

    public function rename(scalar $newName): self
    {
        $this->name = (string)$newName;
    }
}
```

In addition, a similar declaration must allow both disjunction and 
conjunction operators including grouping conditions:

```php
type arrayable = array | (\Traversable & \ArrayAccess & \Countable);

class ArrayHelper
{
    public static function first(arrayable $subject, $default = null)
    {
        if (\is_array($subject)) {
            return $subject[\array_key_first($subject)] ?? $default;
        }
    
        foreach ($subject as $item) {
            return $item;
        }
        
        return $default;
    }
}
```

### Implicit Declaration

Each class or interface declaration implicitly declares a type:

```php
class DatabaseUsersRepository extends Repository implements UsersRepositoryInterface
{
}

//
// Same as implicit type declaration:
//

type DatabaseUsersRepository = Repository | UsersRepositoryInterface | object;
```

```php
class Request implements RequestInterface
{
    use BodyTrait;
    use CookiesTrait;
    use HeadersTrait;
}

//
// Same as implicit type declaration:
//

type Request = RequestInterface | BodyTrait | CookiesTrait | HeadersTrait | object;
```

### Namespace Influence

Types should be affected by namespaces:

```php
namespace App\Helpers;

type scalar = bool | int | float | string | null;
```

```php
namespace App\Console;

class Application
{
    public function __construct(App\Helpers\scalar $name)
    {
    }
}
```

### Namespace Imports

Types allowed to import in the current namespace and the ability to rename:

```php
namespace App;

use type App\Helpers\scalar;
use type callable as alias;
```

### Overriding

TODO reconcile builtin types hierarchy:

```
type mixed = [native code]; // "mixed" type-hint is an alias for type-hint absence

    type object = mixed & [native code];
    
        type \stdClass = object & [native code "instance of"]
        
    type scalar = mixed & [native code];
    
        type string = scalar & [native code];
        
        type int = scalar & [native code];
        
        type float = scalar & [native code];
        
        type bool = scalar & [native code];
        
    type resource = mixed & [native code];
    
    type array = mixed & [native code];
```

Note:
  - From right to left ( <- ) for arguments.
  - From left to right ( -> ) for return types.

```php
interface ParentInterface
{
    public function example(string $message): scalar;
}

//
// @see https://wiki.php.net/rfc/parameter-no-type-variance
//
interface ValidChildAInterface extends ParentInterface
{
    public function example(mixed $message): scalar;
    // public function example($message): scalar;
}

interface ValidChildInterface extends ParentInterface
{
    public function example(scalar $message): string; // OK
}

interface InvalidChildAInterface extends ParentInterface
{
    public function example(int $message): string; // Error: string $message -> int $message
}

interface InvalidChildBInterface extends ParentInterface
{
    public function example(string $message): mixed; // Error: scalar -> mixed retunt type hint
}
```

## Discussion

I would like to offer additional options for the development of this idea

### Runtime Instructions

Because at the language level, type-hint assertions are executed in runtime, 
this does not prevent adding the ability to add runtime (anonymous functions):

```php
type callableArray = array & 
    // Runtime assertion
    fn(array $callable): bool => 
        \count($callable) === 2 && \method_exists(...$callable);

type callableString = string & 
    // Runtime assertion
    fn(string $callable): bool => \function_exists($callable);

type callable = \Closure | callableArray | callableString;
```

I do not know how to make friends with overload =)

## Critique

TBD

## Backward Incompatible Changes

None.  
