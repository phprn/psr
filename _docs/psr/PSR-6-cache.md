---
title: PRS-6
category: PSRs
order: 5
---
| Informações Adicionais                             |
| ---------------------------------------------------|
| [Caching Interface][Caching Interface]             |
| [Cache Meta Document][Cache Meta Document]         |

[Caching Interface]: #Caching Interface
[Cache Meta Document]: #Cache Meta Document

<h1 id="Caching Interface">Caching Interface</h1>

Caching is a common way to improve the performance of any project, making
caching libraries one of the most common features of many frameworks and
libraries. This has lead to a situation where many libraries roll their own
caching libraries, with various levels of functionality. These differences are
causing developers to have to learn multiple systems which may or may not
provide the functionality they need. In addition, the developers of caching
libraries themselves face a choice between only supporting a limited number
of frameworks or creating a large number of adapter classes.

A common interface for caching systems will solve these problems. Library and
framework developers can count on the caching systems working the way they're
expecting, while the developers of caching systems will only have to implement
a single set of interfaces rather than a whole assortment of adapters.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119][].

[RFC 2119]: http://tools.ietf.org/html/rfc2119

## Goal

The goal of this PSR is to allow developers to create cache-aware libraries that
can be integrated into existing frameworks and systems without the need for
custom development.

## Definitions

*    **Calling Library** - The library or code that actually needs the cache
services. This library will utilize caching services that implement this
standard's interfaces, but will otherwise have no knowledge of the
implementation of those caching services.

*    **Implementing Library** - This library is responsible for implementing
this standard in order to provide caching services to any Calling Library. The
Implementing Library MUST provide classes which implement the
Cache\CacheItemPoolInterface and Cache\CacheItemInterface interfaces.
Implementing Libraries MUST support at minimum TTL functionality as described
below with whole-second granularity.

*    **TTL** - The Time To Live (TTL) of an item is the amount of time between
when that item is stored and it is considered stale. The TTL is normally defined
by an integer representing time in seconds, or a DateInterval object.

*    **Expiration** - The actual time when an item is set to go stale. This is
typically calculated by adding the TTL to the time when an object is stored, but
may also be explicitly set with DateTime object. An item with a 300 second TTL
stored at 1:30:00 will have an expiration of 1:35:00. Implementing Libraries MAY
expire an item before its requested Expiration Time, but MUST treat an item as
expired once its Expiration Time is reached. If a calling library asks for an
item to be saved but does not specify an expiration time, or specifies a null
expiration time or TTL, an Implementing Library MAY use a configured default
duration. If no default duration has been set, the Implementing Library MUST
interpret that as a request to cache the item forever, or for as long as the
underlying implementation supports.

*    **Key** - A string of at least one character that uniquely identifies a
cached item. Implementing libraries MUST support keys consisting of the
characters `A-Z`, `a-z`, `0-9`, `_`, and `.` in any order in UTF-8 encoding and a
length of up to 64 characters. Implementing libraries MAY support additional
characters and encodings or longer lengths, but must support at least that
minimum.  Libraries are responsible for their own escaping of key strings
as appropriate, but MUST be able to return the original unmodified key string.
The following characters are reserved for future extensions and MUST NOT be
supported by implementing libraries: `{}()/\@:`

*    **Hit** - A cache hit occurs when a Calling Library requests an Item by key
and a matching value is found for that key, and that value has not expired, and
the value is not invalid for some other reason. Calling Libraries SHOULD make
sure to verify isHit() on all get() calls.

*    **Miss** - A cache miss is the opposite of a cache hit. A cache miss occurs
when a Calling Library requests an item by key and that value not found for that
key, or the value was found but has expired, or the value is invalid for some
other reason. An expired value MUST always be considered a cache miss.

*    **Deferred** - A deferred cache save indicates that a cache item may not be
persisted immediately by the pool. A Pool object MAY delay persisting a deferred
cache item in order to take advantage of bulk-set operations supported by some
storage engines. A Pool MUST ensure that any deferred cache items are eventually
persisted and data is not lost, and MAY persist them before a Calling Library
requests that they be persisted. When a Calling Library invokes the commit()
method all outstanding deferred items MUST be persisted. An Implementing Library
MAY use whatever logic is appropriate to determine when to persist deferred
items, such as an object destructor, persisting all on save(), a timeout or
max-items check or any other appropriate logic. Requests for a cache item that
has been deferred MUST return the deferred but not-yet-persisted item.

## Data

Implementing libraries MUST support all serializable PHP data types, including:

*    **Strings** - Character strings of arbitrary size in any PHP-compatible encoding.
*    **Integers** - All integers of any size supported by PHP, up to 64-bit signed.
*    **Floats** - All signed floating point values.
*    **Boolean** - True and False.
*    **Null** - The actual null value.
*    **Arrays** - Indexed, associative and multidimensional arrays of arbitrary depth.
*    **Object** - Any object that supports lossless serialization and
deserialization such that $o == unserialize(serialize($o)). Objects MAY
leverage PHP's Serializable interface, `__sleep()` or `__wakeup()` magic methods,
or similar language functionality if appropriate.

All data passed into the Implementing Library MUST be returned exactly as
passed. That includes the variable type. That is, it is an error to return
(string) 5 if (int) 5 was the value saved.  Implementing Libraries MAY use PHP's
serialize()/unserialize() functions internally but are not required to do so.
Compatibility with them is simply used as a baseline for acceptable object values.

If it is not possible to return the exact saved value for any reason, implementing
libraries MUST respond with a cache miss rather than corrupted data.

## Key Concepts

### Pool

The Pool represents a collection of items in a caching system. The pool is
a logical Repository of all items it contains.  All cacheable items are retrieved
from the Pool as an Item object, and all interaction with the whole universe of
cached objects happens through the Pool.

### Items

An Item represents a single key/value pair within a Pool. The key is the primary
unique identifier for an Item and MUST be immutable. The Value MAY be changed
at any time.

## Error handling

While caching is often an important part of application performance, it should never
be a critical part of application functionality. Thus, an error in a cache system SHOULD NOT
result in application failure.  For that reason, Implementing Libraries MUST NOT
throw exceptions other than those defined by the interface, and SHOULD trap any errors
or exceptions triggered by an underlying data store and not allow them to bubble.

An Implementing Library SHOULD log such errors or otherwise report them to an
administrator as appropriate.

If a Calling Library requests that one or more Items be deleted, or that a pool be cleared,
it MUST NOT be considered an error condition if the specified key does not exist. The
post-condition is the same (the key does not exist, or the pool is empty), thus there is
no error condition.

## Interfaces

### CacheItemInterface

CacheItemInterface defines an item inside a cache system.  Each Item object
MUST be associated with a specific key, which can be set according to the
implementing system and is typically passed by the Cache\CacheItemPoolInterface
object.

The Cache\CacheItemInterface object encapsulates the storage and retrieval of
cache items. Each Cache\CacheItemInterface is generated by a
Cache\CacheItemPoolInterface object, which is responsible for any required
setup as well as associating the object with a unique Key.
Cache\CacheItemInterface objects MUST be able to store and retrieve any type of
PHP value defined in the Data section of this document.

Calling Libraries MUST NOT instantiate Item objects themselves. They may only
be requested from a Pool object via the getItem() method.  Calling Libraries
SHOULD NOT assume that an Item created by one Implementing Library is
compatible with a Pool from another Implementing Library.

~~~php
<?php

namespace Psr\Cache;

/**
 * CacheItemInterface defines an interface for interacting with objects inside a cache.
 */
interface CacheItemInterface
{
    /**
     * Returns the key for the current cache item.
     *
     * The key is loaded by the Implementing Library, but should be available to
     * the higher level callers when needed.
     *
     * @return string
     *   The key string for this cache item.
     */
    public function getKey();

    /**
     * Retrieves the value of the item from the cache associated with this object's key.
     *
     * The value returned must be identical to the value originally stored by set().
     *
     * If isHit() returns false, this method MUST return null. Note that null
     * is a legitimate cached value, so the isHit() method SHOULD be used to
     * differentiate between "null value was found" and "no value was found."
     *
     * @return mixed
     *   The value corresponding to this cache item's key, or null if not found.
     */
    public function get();

    /**
     * Confirms if the cache item lookup resulted in a cache hit.
     *
     * Note: This method MUST NOT have a race condition between calling isHit()
     * and calling get().
     *
     * @return bool
     *   True if the request resulted in a cache hit. False otherwise.
     */
    public function isHit();

    /**
     * Sets the value represented by this cache item.
     *
     * The $value argument may be any item that can be serialized by PHP,
     * although the method of serialization is left up to the Implementing
     * Library.
     *
     * @param mixed $value
     *   The serializable value to be stored.
     *
     * @return static
     *   The invoked object.
     */
    public function set($value);

    /**
     * Sets the expiration time for this cache item.
     *
     * @param \DateTimeInterface|null $expiration
     *   The point in time after which the item MUST be considered expired.
     *   If null is passed explicitly, a default value MAY be used. If none is set,
     *   the value should be stored permanently or for as long as the
     *   implementation allows.
     *
     * @return static
     *   The called object.
     */
    public function expiresAt($expiration);

    /**
     * Sets the expiration time for this cache item.
     *
     * @param int|\DateInterval|null $time
     *   The period of time from the present after which the item MUST be considered
     *   expired. An integer parameter is understood to be the time in seconds until
     *   expiration. If null is passed explicitly, a default value MAY be used.
     *   If none is set, the value should be stored permanently or for as long as the
     *   implementation allows.
     *
     * @return static
     *   The called object.
     */
    public function expiresAfter($time);

}
~~~

### CacheItemPoolInterface

The primary purpose of Cache\CacheItemPoolInterface is to accept a key from the
Calling Library and return the associated Cache\CacheItemInterface object.
It is also the primary point of interaction with the entire cache collection.
All configuration and initialization of the Pool is left up to an Implementing
Library.

~~~php
<?php

namespace Psr\Cache;

/**
 * CacheItemPoolInterface generates CacheItemInterface objects.
 */
interface CacheItemPoolInterface
{
    /**
     * Returns a Cache Item representing the specified key.
     *
     * This method must always return a CacheItemInterface object, even in case of
     * a cache miss. It MUST NOT return null.
     *
     * @param string $key
     *   The key for which to return the corresponding Cache Item.
     *
     * @throws InvalidArgumentException
     *   If the $key string is not a legal value a \Psr\Cache\InvalidArgumentException
     *   MUST be thrown.
     *
     * @return CacheItemInterface
     *   The corresponding Cache Item.
     */
    public function getItem($key);

    /**
     * Returns a traversable set of cache items.
     *
     * @param string[] $keys
     *   An indexed array of keys of items to retrieve.
     *
     * @throws InvalidArgumentException
     *   If any of the keys in $keys are not a legal value a \Psr\Cache\InvalidArgumentException
     *   MUST be thrown.
     *
     * @return array|\Traversable
     *   A traversable collection of Cache Items keyed by the cache keys of
     *   each item. A Cache item will be returned for each key, even if that
     *   key is not found. However, if no keys are specified then an empty
     *   traversable MUST be returned instead.
     */
    public function getItems(array $keys = array());

    /**
     * Confirms if the cache contains specified cache item.
     *
     * Note: This method MAY avoid retrieving the cached value for performance reasons.
     * This could result in a race condition with CacheItemInterface::get(). To avoid
     * such situation use CacheItemInterface::isHit() instead.
     *
     * @param string $key
     *   The key for which to check existence.
     *
     * @throws InvalidArgumentException
     *   If the $key string is not a legal value a \Psr\Cache\InvalidArgumentException
     *   MUST be thrown.
     *
     * @return bool
     *   True if item exists in the cache, false otherwise.
     */
    public function hasItem($key);

    /**
     * Deletes all items in the pool.
     *
     * @return bool
     *   True if the pool was successfully cleared. False if there was an error.
     */
    public function clear();

    /**
     * Removes the item from the pool.
     *
     * @param string $key
     *   The key to delete.
     *
     * @throws InvalidArgumentException
     *   If the $key string is not a legal value a \Psr\Cache\InvalidArgumentException
     *   MUST be thrown.
     *
     * @return bool
     *   True if the item was successfully removed. False if there was an error.
     */
    public function deleteItem($key);

    /**
     * Removes multiple items from the pool.
     *
     * @param string[] $keys
     *   An array of keys that should be removed from the pool.

     * @throws InvalidArgumentException
     *   If any of the keys in $keys are not a legal value a \Psr\Cache\InvalidArgumentException
     *   MUST be thrown.
     *
     * @return bool
     *   True if the items were successfully removed. False if there was an error.
     */
    public function deleteItems(array $keys);

    /**
     * Persists a cache item immediately.
     *
     * @param CacheItemInterface $item
     *   The cache item to save.
     *
     * @return bool
     *   True if the item was successfully persisted. False if there was an error.
     */
    public function save(CacheItemInterface $item);

    /**
     * Sets a cache item to be persisted later.
     *
     * @param CacheItemInterface $item
     *   The cache item to save.
     *
     * @return bool
     *   False if the item could not be queued or if a commit was attempted and failed. True otherwise.
     */
    public function saveDeferred(CacheItemInterface $item);

    /**
     * Persists any deferred cache items.
     *
     * @return bool
     *   True if all not-yet-saved items were successfully saved or there were none. False otherwise.
     */
    public function commit();
}
~~~

### CacheException

This exception interface is intended for use when critical errors occur,
including but not limited to *cache setup* such as connecting to a cache server
or invalid credentials supplied.

Any exception thrown by an Implementing Library MUST implement this interface.

~~~php
<?php

namespace Psr\Cache;

/**
 * Exception interface for all exceptions thrown by an Implementing Library.
 */
interface CacheException
{
}
~~~

### InvalidArgumentException

~~~php
<?php

namespace Psr\Cache;

/**
 * Exception interface for invalid cache arguments.
 *
 * Any time an invalid argument is passed into a method it must throw an
 * exception class which implements Psr\Cache\InvalidArgumentException.
 */
interface InvalidArgumentException extends CacheException
{
}
~~~

<h1 id="Cache Meta Document">Cache Meta Document</h1>

## 1. Summary

Caching is a common way to improve the performance of any project, making
caching libraries one of the most common features of many frameworks and
libraries. This has lead to a situation where many libraries roll their own
caching libraries, with various levels of functionality. These differences are
causing developers to have to learn multiple systems which may or may not
provide the functionality they need. In addition, the developers of caching
libraries themselves face a choice between only supporting a limited number
of frameworks or creating a large number of adapter classes.

## 2. Why Bother?

A common interface for caching systems will solve these problems. Library and
framework developers can count on the caching systems working the way they're
expecting, while the developers of caching systems will only have to implement
a single set of interfaces rather than a whole assortment of adapters.

Moreover, the implementation presented here is designed for future extensibility.
It allows a variety of internally-different but API-compatible implementations
and offers a clear path for future extension by later PSRs or by specific
implementers.

Pros:
* A standard interface for caching allows free-standing libraries to support
caching of intermediary data without effort; they may simply (optionally) depend
on this standard interface and leverage it without being concerned about
implementation details.
* Commonly developed caching libraries shared by multiple projects, even if
they extend this interface, are likely to be more robust than a dozen separately
developed implementations.

Cons:
* Any interface standardization runs the risk of stifling future innovation as
being "not the Way It's Done(tm)".  However, we believe caching is a sufficiently
commoditized problem space that the extension capability offered here mitigates
any potential risk of stagnation.

## 3. Scope

### 3.1 Goals

* A common interface for basic and intermediate-level caching needs.
* A clear mechanism for extending the specification to support advanced features,
both by future PSRs or by individual implementations. This mechanism must allow
for multiple independent extensions without collision.

### 3.2 Non-Goals

* Architectural compatibility with all existing cache implementations.
* Advanced caching features such as namespacing or tagging that are used by a
minority of users.

## 4. Approaches

### 4.1 Chosen Approach

This specification adopts a "repository model" or "data mapper" model for caching
rather than the more traditional "expire-able key-value" model.  The primary
reason is flexibility.  A simple key/value model is much more difficult to extend.

The model here mandates the use of a CacheItem object, which represents a cache
entry, and a Pool object, which is a given store of cached data.  Items are
retrieved from the pool, interacted with, and returned to it.  While a bit more
verbose at times it offers a good, robust, flexible approach to caching,
especially in cases where caching is more involved than simply saving and
retrieving a string.

Most method names were chosen based on common practice and method names in a
survey of member projects and other popular non-member systems.

Pros:

* Flexible and extensible
* Allows a great deal of variation in implementation without violating the interface
* Does not implicitly expose object constructors as a pseudo-interface.

Cons:

* A bit more verbose than the naive approach

Examples:

Some common usage patterns are shown below.  These are non-normative but should
demonstrate the application of some design decisions.

~~~php
/**
 * Gets a list of available widgets.
 *
 * In this case, we assume the widget list changes so rarely that we want
 * the list cached forever until an explicit clear.
 */
function get_widget_list()
{
    $pool = get_cache_pool('widgets');
    $item = $pool->getItem('widget_list');
    if (!$item->isHit()) {
        $value = compute_expensive_widget_list();
        $item->set($value);
        $pool->save($item);
    }
    return $item->get();
}
~~~

~~~php
/**
 * Caches a list of available widgets.
 *
 * In this case, we assume a list of widgets has been computed and we want
 * to cache it, regardless of what may already be cached.
 */
function save_widget_list($list)
{
    $pool = get_cache_pool('widgets');
    $item = $pool->getItem('widget_list');
    $item->set($list);
    $pool->save($item);
}
~~~

~~~php
/**
 * Clears the list of available widgets.
 *
 * In this case, we simply want to remove the widget list from the cache. We
 * don't care if it was set or not; the post condition is simply "no longer set".
 */
function clear_widget_list()
{
    $pool = get_cache_pool('widgets');
    $pool->deleteItems(['widget_list']);
}
~~~

~~~php
/**
 * Clears all widget information.
 *
 * In this case, we want to empty the entire widget pool. There may be other
 * pools in the application that will be unaffected.
 */
function clear_widget_cache()
{
    $pool = get_cache_pool('widgets');
    $pool->clear();
}
~~~

~~~php
/**
 * Load widgets.
 *
 * We want to get back a list of widgets, of which some are cached and some
 * are not. This of course assumes that loading from the cache is faster than
 * whatever the non-cached loading mechanism is.
 *
 * In this case, we assume widgets may change frequently so we only allow them
 * to be cached for an hour (3600 seconds). We also cache newly-loaded objects
 * back to the pool en masse.
 *
 * Note that a real implementation would probably also want a multi-load
 * operation for widgets, but that's irrelevant for this demonstration.
 */
function load_widgets(array $ids)
{
    $pool = get_cache_pool('widgets');
    $keys = array_map(function($id) { return 'widget.' . $id; }, $ids);
    $items = $pool->getItems($keys);

    $widgets = array();
    foreach ($items as $key => $item) {
        if ($item->isHit()) {
            $value = $item->get();
        } else {
            $value = expensive_widget_load($id);
            $item->set($value);
            $item->expiresAfter(3600);
            $pool->saveDeferred($item, true);
        }
        $widget[$value->id()] = $value;
    }
    $pool->commit(); // If no items were deferred this is a no-op.

    return $widgets;
}
~~~

~~~php
/**
 * This examples reflects functionality that is NOT included in this
 * specification, but is shown as an example of how such functionality MIGHT
 * be added by extending implementations.
 */

interface TaggablePoolInterface extends Psr\Cache\CachePoolInterface
{
    /**
     * Clears only those items from the pool that have the specified tag.
     */
    clearByTag($tag);
}

interface TaggableItemInterface extends Psr\Cache\CacheItemInterface
{
    public function setTags(array $tags);
}

/**
 * Caches a widget with tags.
 */
function set_widget(TaggablePoolInterface $pool, Widget $widget)
{
    $key = 'widget.' . $widget->id();
    $item = $pool->getItem($key);

    $item->setTags($widget->tags());
    $item->set($widget);
    $pool->save($item);
}
~~~

### 4.2 Alternative: "Weak item" approach

A variety of earlier drafts took a simpler "key value with expiration" approach,
also known as a "weak item" approach.  In this model, the "Cache Item" object
was really just a dumb array-with-methods object.  Users would instantiate it
directly, then pass it to a cache pool.  While more familiar, that approach
effectively prevented any meaningful extension of the Cache Item.  It effectively
made the Cache Item's constructor part of the implicit interface, and thus
severely curtailed extensibility or the ability to have the cache item be where
the intelligence lives.

In a poll conducted in June 2013, most participants showed a clear preference for
the more robust if less conventional "Strong item" / repository approach, which
was adopted as the way forward.

Pros:
* More traditional approach.

Cons:
* Less extensible or flexible.

### 4.3 Alternative: "Naked value" approach

Some of the earliest discussions of the Cache spec suggested skipping the Cache
Item concept all together and just reading/writing raw values to be cached.
While simpler, it was pointed out that made it impossible to tell the difference
between a cache miss and whatever raw value was selected to represent a cache
miss.  That is, if a cache lookup returned NULL it's impossible to tell if there
was no cached value or if NULL was the value that had been cached.  (NULL is a
legitimate value to cache in many cases.)

Most more robust caching implementations we reviewed -- in particular the Stash
caching library and the home-grown cache system used by Drupal -- use some sort
of structured object on `get` at least to avoid confusion between a miss and a
sentinel value.  Based on that prior experience FIG decided that a naked value
on `get` was impossible.

### 4.4 Alternative: ArrayAccess Pool

There was a suggestion to make a Pool implement ArrayAccess, which would allow
for cache get/set operations to use array syntax.  That was rejected due to
limited interest, limited flexibility of that approach (trivial get and set with
default control information is all that's possible), and because it's trivial
for a particular implementation to include as an add-on should it desire to
do so.

## 5. People

### 5.1 Editor

* Larry Garfield

### 5.2 Sponsors

* Paul Dragoonis, PPI Framework (Coordinator)
* Robert Hafner, Stash

## 6. Votes

[Acceptance vote on the mailing list](https://groups.google.com/forum/#!msg/php-fig/dSw5IhpKJ1g/O9wpqizWAwAJ)

## 7. Relevant Links

_**Note:** Order descending chronologically._

* [Survey of existing cache implementations][1], by @dragoonis
* [Strong vs. Weak informal poll][2], by @Crell
* [Implementation details informal poll][3], by @Crell

[1]: https://docs.google.com/spreadsheet/ccc?key=0Ak2JdGialLildEM2UjlOdnA4ekg3R1Bfeng5eGlZc1E#gid=0
[2]: https://docs.google.com/spreadsheet/ccc?key=0AsMrMKNHL1uGdDdVd2llN1kxczZQejZaa3JHcXA3b0E#gid=0
[3]: https://docs.google.com/spreadsheet/ccc?key=0AsMrMKNHL1uGdEE3SU8zclNtdTNobWxpZnFyR0llSXc#gid=1

## 8. Errata

### 8.1 Handling of incorrect DateTime values in expiresAt()

The `CacheItemInterface::expiresAt()` method's `$expiration` parameter is untyped
in the interface, but in the docblock is specified as `\DateTimeInterface`.  The
intent is that either a `\DateTime` or `\DateTimeImmutable` object is allowed.
However, `\DateTimeInterface` and `\DateTimeImmutable` were added in PHP 5.5, and
the authors chose not to impose a hard syntactic requirement for PHP 5.5 on the
specification.

Despite that, implementers MUST accept only `\DateTimeInterface` or compatible types
(such as `\DateTime` and `\DateTimeImmutable`) as if the method was explicitly typed.
(Note that the variance rules for a typed parameter may vary between language versions.)

Simulating a failed type check unfortunately varies between PHP versions and thus is not
recommended.  Instead, implementors SHOULD throw an instance of `\Psr\Cache\InvalidArgumentException`.  
The following sample code is recommended in order to enforce the type check on the expiresAt()
method:

```php

class ExpiresAtInvalidParameterException implements Psr\Cache\InvalidArgumentException {}

// ...

if (! (
        null === $expiration
        || $expiration instanceof \DateTime
        || $expiration instanceof \DateTimeInterface
)) {
    throw new ExpiresAtInvalidParameterException(sprintf(
        'Argument 1 passed to %s::expiresAt() must be an instance of DateTime or DateTimeImmutable; %s given',
        get_class($this),
        is_object($expiration) ? get_class($expiration) : gettype($expiration)
    ));
}
```
