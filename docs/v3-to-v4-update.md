# Updating from v3 to v4

Release 4.0.0 raises the PHP requirement and adds return types to several `DataSet` methods. One of
them narrows what the method may return.

## At a glance

| | v3 (3.0.2) | v4 (4.0.0) |
|---|---|---|
| PHP | `^8.1.0` | `^8.3.0` |
| `DataSet::key()` | untyped | `: int\|bool\|null` — **breaks string keys** (fixed in the next 4.x release) |
| `DataSet::keys()` | untyped | `: array` |
| `DataSet::walk()` | untyped | `: bool` |
| `DataObject::getIterator()` | returned the dump directly | goes through `ArrayHelper::fromObject()` |

## Minimum supported PHP version raised

All Framework packages now require **PHP 8.3** or newer.

## `DataSet::key()` lost string keys in 4.0.0

```php
// v3
#[\ReturnTypeWillChange]
public function key()

// v4.0.0
public function key(): int|bool|null

// later 4.x releases
public function key(): int|string|bool|null
```

`DataSet` accepts any offset — `offsetSet()` does no key check, and the constructor keeps the keys
of the array it is given. A set built with string keys therefore iterates fine in 3.x, but in 4.0.0
the return type coerces every string key to `true`:

```php
$set = new DataSet(['first' => new DataObject(['a' => 1])]);

foreach ($set as $key => $object) {
    // v4.0.0: $key === true
    // later 4.x releases: $key === 'first'
}
```

In later 4.x releases, `key()` also allows `string` and iteration yields the original keys again. A subclass
that overrides `key()` with the 4.0.0 signature `int|bool|null` still works, but loses string keys
the same way; widen its return type to `int|string|bool|null`.

If you have to support 4.0.0, either switch to integer keys:

```php
$set = new DataSet(array_values($objects));
```

or iterate over `keys()` instead of the object itself:

```php
foreach ($set->keys() as $key) {
    $object = $set[$key];
}
```

## `DataObject::getIterator()` normalises through `ArrayHelper`

```php
// v3
return new \ArrayIterator($this->dump(0));

// v4
$value = ArrayHelper::fromObject($this->dump(0));

return new \ArrayIterator($value);
```

`dump(0)` returns a `stdClass`; `ArrayHelper::fromObject()` turns it into an array. Iterating a
`DataObject` yielded the same key/value pairs before, so this is a correctness fix rather than a
behaviour change for callers.

> This adds a code dependency on `joomla/utilities`, which is **not listed** in this package's
> `require`. It resolves today only because `joomla/registry` pulls it in. Add it explicitly if you
> depend on `getIterator()`:
>
> ```bash
> composer require joomla/utilities
> ```

## `keys()` and `walk()` typed

`keys(): array` and `walk(callable $funcname): bool` match what both already returned. An override
in a subclass must now declare the same types.

## Dependency changes

| Package | v3 (3.0.2) | v4 (4.0.0) |
|---|---|---|
| `php` | `^8.1.0` | `^8.3.0` |
| `joomla/registry` | `^3.0` | `^4.0` |
