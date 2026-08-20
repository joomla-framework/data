# Data package

### `Data\DataObject`

`Data\DataObject` is a class that is used to store data but allowing you to access the data by mimicking the way PHP handles class properties. Rather than explicitly declaring properties in the class, `Data\DataObject` stores virtual properties of the class in a private internal array. Concrete properties can still be defined but these are separate from the data.

#### Construction

The constructor for a new `Data\DataObject` object can optionally take an array or an object. The keys of the array or the properties of the object will be bound to the properties of the `Data\DataObject` object.

```php
use Joomla\Data\DataObject;

// Create an empty object.
$object1 = new DataObject;

// Create an object with data. You can use an array or another object.
$data = array(
    'foo' => 'bar',
);

$object2 = new DataObject($data);

// The following should echo "bar".
echo $object2->foo;
```

#### General Usage

`Data\DataObject` includes magic getters and setters to provide access to the internal property store as if they were explicitly declared properties of the class.

The `bind` method allows for injecting an existing array or object into the `Data\DataObject` object.

The `dump` method gets a plain `stdClass` version of the `Data\DataObject` object's properties. It will also support recursion to a specified number of levels where the default is 3 and a depth of 0 would return a `stdClass` object with all the properties in native form. Note that the `dump` method will only return virtual properties set binding and magic methods. It will not include any concrete properties defined in the class itself.

The `JsonSerializable` interface is implemented. This method proxies to the `dump` method (defaulting to a recursion depth of 3). Note that this interface only takes effect implicitly in PHP 5.4 so any code built for PHP 5.3 needs to explicitly use either the `jsonSerialize` or the `dump` method before passing to `json_encode`.

The `Data\DataObject` class also implements the `IteratorAggregate` interface so it can easily be used in a `foreach` statement.

```php
use Joomla\Data\DataObject;

// Create an empty object.
$object = new DataObject;

// Set a property.
$object->foo = 'bar';

// Get a property.
$foo = $object->foo;

// Binding some new data to the object.
$object->bind(array('goo' => 'car');

// Get a plain object version of the data object.
$stdClass = $object->dump();

// Get a property with a default value if it is not already set.
$foo = $object->foo ?: 'The default';

// Iterate over the properties as if the object were a real array.
foreach ($object as $key => $value)
{
    echo "\n$key = $value";
}

if (version_compare(PHP_VERSION, '5.4') >= 0)
{
	// PHP 5.4 is aware of the JsonSerializable interface.
	$json = json_encode($object);
}
else
{
	// Have to do it the hard way to be compatible with PHP 5.3.
	$json = json_encode($object->jsonSerialize());
}
```

### `Data\DataSet`

`Data\DataSet` is a collection class that allows the developer to operate on a list of `Data\DataObject` objects as if they were in a typical PHP array (`Data\DataSet` implements the `ArrayAccess`, `Countable` and `Iterator` interfaces).

#### Construction

A typical `Data\DataSet` object will be instantiated by passing an array of `Data\DataObject` objects in the constructor.

```php
use Joomla\Data\DataObject;
use Joomla\Data\DataSet;

// Create an empty object.
$players = new DataSet(
    array(
        new DataObject(array('race' => 'Elf', 'level' => 1)),
        new DataObject(array('race' => 'Chaos Dwarf', 'level' => 2)),
    )
);
```

#### General Usage

Array elements can be manipulated with the `offsetSet` and `offsetUnset` methods, or by using PHP array nomenclature.

The magic `__get` method in the `Data\DataSet` class effectively works like a "get column" method. It will return an array of values of the properties for all the objects in the list.

The magic `__set` method is similar and works like a "set column" method. It will set all a value for a property for all the objects in the list.

The `clear` method will clear all the objects in the data set.

The `keys` method will return all of the keys of the objects stored in the set. It works like the `array_keys` function does on an PHP array.

```php
use Joomla\Data\DataObject;

// Add a new element to the end of the list.
$players[] => new DataObject(array('race' => 'Skaven', 'level' => 2));

// Add a new element with an associative key.
$players['captain'] => new DataObject(array('race' => 'Human', 'level' => 3));

// Get a keyed element from the list.
$captain = $players['captain'];

// Set the value of a property for all objects. Upgrade all players to level 4.
$players->level = 4;

// Get the value of a property for all object and also the count (get the average level).
$average = $players->level / count($players);

// Clear all the objects.
$players->clear();
```

`Data\DataSet` supports magic methods that operate on all the objects in the list. Calling an arbitrary method will iterate of the list of objects, checking if each object has a callable method of the name of the method that was invoked. In such a case, the return values are assembled in an array forming the return value of the method invoked on the `Data\DataSet` object. The keys of the original objects are maintained in the result array.

```php
use Joomla\Data\DataObject;
use Joomla\Data\DataSet;

/**
 * A custom data object.
 *
 * @since  1.0
 */
class PlayerObject extends DataObject
{
    /**
     * Get player damage.
     *
     * @return  integer  The amount of damage the player has received.
     *
     * @since   1.0
     */
    public function hurt()
    {
        return (int) $this->maxHealth - $this->actualHealth;
    }
}

$players = new DataSet(
    array(
        // Add a normal player.
        new PlayerObject(array('race' => 'Chaos Dwarf', 'level' => 2,
        	'maxHealth' => 40, 'actualHealth' => '32')),
        // Add an invincible player.
        new PlayerObject(array('race' => 'Elf', 'level' => 1)),
    )
);

// Get an array of the hurt players.
$hurt = $players->hurt();

if (!empty($hurt))
{
    // In this case, $hurt = array(0 => 8);
    // There is no entry for the second player
    // because that object does not have a "hurt" method.
    foreach ($hurt as $playerKey => $player)
    {
        // Do something with the hurt players.
    }
};
```

### `Data\DumpableInterface`

`Data\DumpableInterface` is an interface that defines a `dump` method for dumping the properties of an object as a `stdClass` with or without recursion.

## Things to know before you build on this

**`toArray()` on an empty `DataSet` throws.** `getObjectsKeys()` starts with `$keys = null` and
only assigns inside the loop, so an empty set reaches `array_keys(null)` and raises a `TypeError`.
Since `toArray()` calls it whenever no explicit key list is given, this is the empty result set —
the most ordinary case there is:

```php
(new DataSet())->toArray();   // TypeError
```

Guard before calling, or pass the keys explicitly:

```php
$rows = count($set) ? $set->toArray() : [];
$rows = $set->toArray(true, 'id', 'title');
```

**Circular references overflow on `json_encode()`.** `dump()` guards against cycles within one
call, but leaves the already-seen object in place rather than replacing it with a marker.
`json_encode()` then calls that object's `jsonSerialize()`, which starts a fresh `dump()` with an
empty tracker, and the two objects bounce until memory runs out:

```php
$a = new DataObject(); $b = new DataObject();
$a->b = $b; $b->a = $a;

json_encode($a);   // exhausts memory
```

Break the cycle before serialising, or serialise a projection instead of the object graph.

**`isset()` and `bind()` disagree about null.** `bind(['a' => null])` stores the property, but
`__isset()` checks `isset($this->properties['a'])`, which is `false` for a null value. There is no
`hasProperty()` to tell "absent" from "present but null" apart.

**`__isset()` and `__unset()` bypass the extension points.** `__get()` and `__set()` delegate to
the protected `getProperty()`/`setProperty()`, but `__isset()` and `__unset()` read and write
`$this->properties` directly. A subclass that overrides `getProperty()` — for computed properties,
say — will find `isset($obj->computed)` returning `false` while `$obj->computed` works.

**`DataSet` uses a single internal cursor.** It implements `Iterator` rather than
`IteratorAggregate`, so two nested `foreach` loops over the same set interfere: the inner loop
advances the cursor the outer one is using.

**Key extraction round-trips through JSON.** `getObjectsKeys()` runs `json_decode(json_encode($object), true)`
per object. That is slow for large sets, and a value containing invalid UTF-8 makes `json_encode()`
return `false`, after which the method fails the same way as the empty-set case above.
