# Updating from v4 to v5

Release 5.0.0 turns `DataSet` into an `IteratorAggregate`, makes calls of unknown methods on a
`DataSet` fail loudly, and adds native parameter and return types to every method of the package.

## At a glance

| | v4 | v5 |
|---|---|---|
| `DataSet` iteration | `Iterator` with an internal cursor | `IteratorAggregate`, `getIterator(): \ArrayIterator` |
| `DataSet::current()`, `key()`, `next()`, `rewind()`, `valid()` | public | **removed** |
| `DataSet::__call()` with a method no object has | returns `[]` | throws `\BadMethodCallException` |
| `DataSet::walk()` | returns `true` | returns the set (`static`) |
| `DataObject::bind()`/constructor with a scalar | `\InvalidArgumentException` | `\TypeError` |
| Parameter and return types | mostly doc blocks, `#[\ReturnTypeWillChange]` | native types everywhere |

## `DataSet` is an `IteratorAggregate`

```php
// v4
class DataSet implements DumpableInterface, \ArrayAccess, \Countable, \Iterator

// v5
class DataSet implements DumpableInterface, \ArrayAccess, \Countable, \IteratorAggregate
```

`getIterator()` returns a new `\ArrayIterator` over the objects on every call. Nested `foreach` loops
over the same set now work, and unsetting objects inside a `foreach` no longer needs the special
handling the cursor had.

The cursor methods `current()`, `key()`, `next()`, `rewind()` and `valid()` are gone. Code that
called them directly has to use an iterator instead:

```php
// v4
$set->rewind();
$first = $set->current();

// v5
$iterator = $set->getIterator();
$first    = $iterator->current();

// or, without an iterator
$keys  = $set->keys();
$first = $keys === [] ? null : $set[$keys[0]];
```

A subclass that overrides one of the removed methods must drop the override or move the logic into
`getIterator()`.

## `DataSet::__call()` throws for unknown methods

```php
$set = new DataSet([new DataObject()]);

// v4: []
// v5: BadMethodCallException: Call to undefined method Joomla\Data\DataSet::misspelt()
$set->misspelt();
```

The exception is thrown only when the set is not empty and **none** of its objects has a callable
method of that name. Objects without the method are still skipped when at least one object has it,
and an empty set still returns `[]`.

## `DataSet::walk()` returns the set

`walk()` returned `true` unconditionally. It now returns the set itself, like `clear()`, so calls can
be chained. Code that tested the return value (`if ($set->walk(...))`) keeps working, because an
object is truthy; code that compared it strictly (`=== true`) does not.

## Native types

Every method now declares native parameter and return types, and the `#[\ReturnTypeWillChange]`
attributes are gone. The types match what the doc blocks already said:

| Method | v5 signature |
|---|---|
| `DumpableInterface::dump()` | `dump(int $depth = 3, ?\SplObjectStorage $dumped = null): \stdClass\|array` |
| `DataObject::__construct()` | `__construct(array\|object $properties = [])` |
| `DataObject::__get()` | `__get(string $property): mixed` |
| `DataObject::__isset()` | `__isset(string $property): bool` |
| `DataObject::__set()` | `__set(string $property, mixed $value): void` |
| `DataObject::__unset()` | `__unset(string $property): void` |
| `DataObject::bind()` | `bind(array\|object $properties, bool $updateNulls = true): static` |
| `DataObject::dump()` | `dump(int $depth = 3, ?\SplObjectStorage $dumped = null): \stdClass` |
| `DataObject::getIterator()` | `getIterator(): \ArrayIterator` |
| `DataObject::jsonSerialize()` | `jsonSerialize(): \stdClass` |
| `DataObject::count()` | `count(): int` |
| `DataObject::dumpProperty()` (protected) | `dumpProperty(string $property, int $depth, \SplObjectStorage $dumped): mixed` |
| `DataObject::getProperty()` (protected) | `getProperty(string $property): mixed` |
| `DataObject::setProperty()` (protected) | `setProperty(string $property, mixed $value): mixed` |
| `DataSet::__call()` | `__call(string $method, array $arguments = []): array` |
| `DataSet::__get()` | `__get(string $property): array` |
| `DataSet::__isset()` | `__isset(string $property): bool` |
| `DataSet::__set()` | `__set(string $property, mixed $value): void` |
| `DataSet::__unset()` | `__unset(string $property): void` |
| `DataSet::getObjectsKeys()` | `getObjectsKeys(string $type = 'all'): array` |
| `DataSet::toArray()` | `toArray(bool $associative = true, string ...$keys): array` |
| `DataSet::count()` | `count(): int` |
| `DataSet::clear()` | `clear(): static` |
| `DataSet::dump()` | `dump(int $depth = 3, ?\SplObjectStorage $dumped = null): array` |
| `DataSet::jsonSerialize()` | `jsonSerialize(): array` |
| `DataSet::offsetExists()`, `offsetGet()`, `offsetSet()`, `offsetUnset()` | `mixed $offset` (and `mixed $object` for `offsetSet()`) |

What this means for you:

* **Subclasses** that override any of these methods must declare compatible types. In particular,
  the common pattern of overriding the protected `setProperty()`/`getProperty()` needs the return
  type now:

  ```php
  // v4
  protected function setProperty($property, $value)

  // v5
  protected function setProperty(string $property, mixed $value): mixed
  ```

  `DumpableInterface` implementations outside this package need the typed `dump()` signature.
* **`bind()` and the constructor** no longer check the argument themselves. Passing a scalar raises
  a `\TypeError` instead of an `\InvalidArgumentException`.
* `setProperty()` returns `null` (instead of nothing) when it discards a property name starting with
  a null byte; the observable result is the same.

`DataSet::offsetSet()` keeps `mixed $object` because `ArrayAccess` requires it; it still throws
`\InvalidArgumentException` for anything that is not a `DataObject`.
