---
layout: documentation
title: Working with Advanced Column Types
---

# Working with Advanced Column Types #

Propel offers a set of advanced column types. The database-agnostic implementation allows these types to work on all supported RDBMS.

## `blob` Columns ##

Propel uses PHP streams internally for storing _Binary_ Locator Objects (BLOBs).  This choice was made because PDO itself uses streams as a convention when returning LOB columns in a resultset and when binding values to prepared statements.  Unfortunately, not all PDO drivers support this (see, for example, [http://bugs.php.net/bug.php?id=40913](http://bugs.php.net/bug.php?id=40913)); in those cases, Propel creates a `php://temp` stream to hold the LOB contents and thus provide a consistent API.

Note that CLOB (_Character_ Locator Objects) are treated as strings in Propel, as there is no convention for them to be treated as streams by PDO.

### Getting `blob` Values ###

BLOB values will be returned as PHP stream resources from the accessor methods.  Alternatively, if the value is NULL in the database, then the accessors will return the PHP value NULL.

```php
<?php
$media = MediaQuery::create()->findPk(1);
$fp = $media->getCoverImage();
if ($fp !== null) {
  echo stream_get_contents($fp);
}
```

### Setting `blob` Values ###

When setting a blob column, you can either pass in a stream or the blob contents.

```php
<?php
// Setting using a stream
$fp = fopen("/path/to/file.ext", "rb");
$media = new Media();
$media->setCoverImage($fp);

// Setting using file contents
$media = new Media();
$media->setCoverImage(file_get_contents("/path/to/file.ext"));
```

Regardless of which setting method you choose, the blob will always be represented internally as a stream resource -- _and subsequent calls to the accessor methods will return a stream._

For example:
```php
<?php
$media = new Media();
$media->setCoverImage(file_get_contents("/path/to/file.ext"));

$fp = $media->getCoverImage();
print gettype($fp); // "resource"
```

### Setting `blob` columns and isModified() ###

Note that because a stream contents may be externally modified, _mutator methods for blob columns will always set the _isModified()_ to report true_ -- even if the stream has the same identity as the stream that was returned.

For example:
```php
<?php

$media = MediaQuery::create()->findPk(1);
$fp = $media->getCoverImage();
$media->setCoverImage($fp);

var_export($media->isModified()); // TRUE
```

## `enum` Columns ##

Although stored in the database as integers, enum columns let users manipulate a set of predefined values, without worrying about their storage.

```xml
<table name="book">
  ...
  <column name="style" type="enum" valueSet="novel, essay, poetry" />
</table>
```

```php
<?php
// The ActiveRecord setter and getter let users use any value from the valueSet
$book = new Book();
$book->setStyle('novel');
echo $book->getStyle(); // novel
// Trying to set a value not in the valueSet throws an exception

// Enum columns are also searchable, using the generated filterByXXX() method
// or other ModelCriteria methods (like where(), condition())
$books = BookQuery::create()
  ->filterByStyle('novel')
  ->find();
```

## `object` Columns ##

Propel offers an `object` column type to store PHP objects in the database. The column setter serializes the object, which is later stored to the database as a string. The column getter unserializes the string and returns the object. Therefore, for the end user, the column contains an object.

### Getting and Setting `object` Values ###

```php
<?php
class GeographicCoordinates
{
  public $latitude, $longitude;

  public function __construct($latitude, $longitude)
  {
    $this->latitude = $latitude;
    $this->longitude = $longitude;
  }

  public function isInNorthernHemisphere()
  {
    return $this->latitude > 0;
  }
}

// The 'house' table has a 'coordinates' column of type object
$house = new House();
$house->setCoordinates(new GeographicCoordinates(48.8527, 2.3510));
echo $house->getCoordinates()->isInNorthernHemisphere(); // true
$house->save();
```

### Retrieving Records based on `object` Values ###

Not only do `object` columns benefit from these smart getter and setter in the generated Active Record class, they are also searchable using the generated `filterByXXX()` method in the query class:

```php
<?php
$house = HouseQuery::create()
 ->filterByCoordinates(new GeographicCoordinates(48.8527, 2.3510))
 ->find();
```

Propel looks in the database for a serialized version of the object passed as parameter of the `filterByXXX()` method.

## `array` Columns ##

An `array` column can store a simple PHP array in the database (nested arrays and associative arrays are not accepted). The column setter serializes the array, which is later stored to the database as a string. The column getter unserializes the string and returns the array. Therefore, for the end user, the column contains an array.

### Getting and Setting `array` Values ###

```php
<?php
// The 'book' table has a 'tags' column of type array
$book = new Book();
$book->setTags(array('novel', 'russian'));
print_r($book->getTags()); // array('novel', 'russian')

// If the column name is plural, Propel also generates hasXXX(), addXXX(),
// and removeXXX() methods, where XXX is the singular column name
echo $book->hasTag('novel'); // true
$book->addTag('romantic');
print_r($book->getTags()); // array('novel', 'russian', 'romantic')
$book->removeTag('russian');
print_r($book->getTags()); // array('novel', 'romantic')
```

### Retrieving Records based on `array` Values ###

Propel doesn't use `serialize()` to transform the array into a string. Instead, it uses a special serialization function, that makes it possible to search for values of `array` columns.

```php
<?php
// Search books that contain all the specified tags
$books = BookQuery::create()
  ->filterByTags(array('novel', 'russian'), Criteria::CONTAINS_ALL)
  ->find();

// Search books that contain at least one of the specified tags
$books = BookQuery::create()
  ->filterByTags(array('novel', 'russian'), Criteria::CONTAINS_SOME)
  ->find();

// Search books that don't contain any of the specified tags
$books = BookQuery::create()
  ->filterByTags(array('novel', 'russian'), Criteria::CONTAINS_NONE)
  ->find();

// If the column name is plural, Propel also generates singular filter methods
// expecting a scalar parameter instead of an array
$books = BookQuery::create()
  ->filterByTag('russian')
  ->find();
```

>**Tip**Filters on array columns translate to SQL as LIKE conditions. That means that the resulting query often requires a full table scan, and is not suited for large tables.

_Warning_: Only generated Query classes (through generated `filterByXXX()` methods) and `ModelCriteria` (through `where()`, and `condition()`) allow conditions on `enum`, `object`, and `array` columns. `Criteria` alone (through `add()`, `addAnd()`, and `addOr()`) does not support conditions on such columns.
