---
title: PRS-4
category: PSRs
order: 4
---

| Informações Adicionais                             |
| ---------------------------------------------------|
| [Autoloader][Autoloader]                           |
| [Meta Document][Meta Document]                     |
| [Example Implementations][Example Implementations] |

[Autoloader]: #Autoloader
[Meta Document]: #Meta Document
[Example Implementations]: #Example Implementations


<h1 id="Autoloader">Autoloader</h1>


The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

## 1. Overview

This PSR describes a specification for [autoloading][] classes from file
paths. It is fully interoperable, and can be used in addition to any other
autoloading specification, including [PSR-0][]. This PSR also describes where
to place files that will be autoloaded according to the specification.

## 2. Specification

1. The term "class" refers to classes, interfaces, traits, and other similar
   structures.

2. A fully qualified class name has the following form:

        \<NamespaceName>(\<SubNamespaceNames>)*\<ClassName>

    1. The fully qualified class name MUST have a top-level namespace name,
       also known as a "vendor namespace".

    2. The fully qualified class name MAY have one or more sub-namespace
       names.

    3. The fully qualified class name MUST have a terminating class name.

    4. Underscores have no special meaning in any portion of the fully
       qualified class name.

    5. Alphabetic characters in the fully qualified class name MAY be any
       combination of lower case and upper case.

    6. All class names MUST be referenced in a case-sensitive fashion.

3. When loading a file that corresponds to a fully qualified class name ...

    1. A contiguous series of one or more leading namespace and sub-namespace
       names, not including the leading namespace separator, in the fully
       qualified class name (a "namespace prefix") corresponds to at least one
       "base directory".

    2. The contiguous sub-namespace names after the "namespace prefix"
       correspond to a subdirectory within a "base directory", in which the
       namespace separators represent directory separators. The subdirectory
       name MUST match the case of the sub-namespace names.

    3. The terminating class name corresponds to a file name ending in `.php`.
       The file name MUST match the case of the terminating class name.

4. Autoloader implementations MUST NOT throw exceptions, MUST NOT raise errors
   of any level, and SHOULD NOT return a value.

## 3. Examples

The table below shows the corresponding file path for a given fully qualified
class name, namespace prefix, and base directory.

| Fully Qualified Class Name    | Namespace Prefix   | Base Directory           | Resulting File Path
| ----------------------------- |--------------------|--------------------------|-------------------------------------------
| \Acme\Log\Writer\File_Writer  | Acme\Log\Writer    | ./acme-log-writer/lib/   | ./acme-log-writer/lib/File_Writer.php
| \Aura\Web\Response\Status     | Aura\Web           | /path/to/aura-web/src/   | /path/to/aura-web/src/Response/Status.php
| \Symfony\Core\Request         | Symfony\Core       | ./vendor/Symfony/Core/   | ./vendor/Symfony/Core/Request.php
| \Zend\Acl                     | Zend               | /usr/includes/Zend/      | /usr/includes/Zend/Acl.php

For example implementations of autoloaders conforming to the specification,
please see the [examples file][]. Example implementations MUST NOT be regarded
as part of the specification and MAY change at any time.

[autoloading]: http://php.net/autoload
[PSR-0]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md
[examples file]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader-examples.md


<h1 id="Meta Document">PSR-4 Meta Document</h1>

## 1. Summary

The purpose is to specify the rules for an interoperable PHP autoloader that
maps namespaces to file system paths, and that can co-exist with any other SPL
registered autoloader.  This would be an addition to, not a replacement for,
PSR-0.

## 2. Why Bother?

### History of PSR-0

The PSR-0 class naming and autoloading standard rose out of the broad
acceptance of the Horde/PEAR convention under the constraints of PHP 5.2 and
previous. With that convention, the tendency was to put all PHP source classes
in a single main directory, using underscores in the class name to indicate
pseudo-namespaces, like so:

    /path/to/src/
        VendorFoo/
            Bar/
                Baz.php     # VendorFoo_Bar_Baz
        VendorDib/
            Zim/
                Gir.php     # Vendor_Dib_Zim_Gir

With the release of PHP 5.3 and the availability of namespaces proper, PSR-0
was introduced to allow both the old Horde/PEAR underscore mode *and* the use
of the new namespace notation. Underscores were still allowed in the class
name to ease the transition from the older namespace naming to the newer naming,
and thereby to encourage wider adoption.

    /path/to/src/
        VendorFoo/
            Bar/
                Baz.php     # VendorFoo_Bar_Baz
        VendorDib/
            Zim/
                Gir.php     # VendorDib_Zim_Gir
        Irk_Operation/
            Impending_Doom/
                V1.php
                V2.php      # Irk_Operation\Impending_Doom\V2

This structure is informed very much by the fact that the PEAR installer moved
source files from PEAR packages into a single central directory.

### Along Comes Composer

With Composer, package sources are no longer copied to a single global
location. They are used from their installed location and are not moved
around. This means that with Composer there is no "single main directory" for
PHP sources as with PEAR. Instead, there are multiple directories; each
package is in a separate directory for each project.

To meet the requirements of PSR-0, this leads to Composer packages looking
like this:

    vendor/
        vendor_name/
            package_name/
                src/
                    Vendor_Name/
                        Package_Name/
                            ClassName.php       # Vendor_Name\Package_Name\ClassName
                tests/
                    Vendor_Name/
                        Package_Name/
                            ClassNameTest.php   # Vendor_Name\Package_Name\ClassNameTest

The "src" and "tests" directories have to include vendor and package directory
names. This is an artifact of PSR-0 compliance.

Many find this structure to be deeper and more repetitive than necessary. This
proposal suggests that an additional or superseding PSR would be useful so
that we can have packages that look more like the following:

    vendor/
        vendor_name/
            package_name/
                src/
                    ClassName.php       # Vendor_Name\Package_Name\ClassName
                tests/
                    ClassNameTest.php   # Vendor_Name\Package_Name\ClassNameTest

This would require an implementation of what was initially called
"package-oriented autoloading" (as vs the traditional "direct class-to-file
autoloading").

### Package-Oriented Autoloading

It's difficult to implement package-oriented autoloading via an extension or
amendment to PSR-0, because PSR-0 does not allow for an intercessory path
between any portions of the class name. This means the implementation of a
package-oriented autoloader would be more complicated than PSR-0. However, it
would allow for cleaner packages.

Initially, the following rules were suggested:

1. Implementors MUST use at least two namespace levels: a vendor name, and
package name within that vendor. (This top-level two-name combination is
hereinafter referred to as the vendor-package name or the vendor-package
namespace.)

2. Implementors MUST allow a path infix between the vendor-package namespace
and the remainder of the fully qualified class name.

3. The vendor-package namespace MAY map to any directory. The remaining
portion of the fully-qualified class name MUST map the namespace names to
identically-named directories, and MUST map the class name to an
identically-named file ending in .php.

Note that this means the end of underscore-as-directory-separator in the class
name. One might think underscores should be honored as they are under
PSR-0, but seeing as their presence in that document is in reference to
transitioning away from PHP 5.2 and previous pseudo-namespacing, it is
acceptable to remove them here as well.

## 3. Scope

### 3.1 Goals

- Retain the PSR-0 rule that implementors MUST use at least two namespace
  levels: a vendor name, and package name within that vendor.

- Allow a path infix between the vendor-package namespace and the remainder of
  the fully qualified class name.

- Allow the vendor-package namespace MAY map to any directory, perhaps
  multiple directories.

- End the honoring of underscores in class names as directory separators

### 3.2 Non-Goals

- Provide a general transformation algorithm for non-class resources

## 4. Approaches

### 4.1 Chosen Approach

This approach retains key characteristics of PSR-0 while eliminating the
deeper directory structures it requires. In addition, it specifies certain
additional rules that make implementations explicitly more interoperable.

Although not related to directory mapping, the final draft also specifies how
autoloaders should handle errors.  Specifically, it forbids throwing exceptions
or raising errors.  The reason is two-fold.

1. Autoloaders in PHP are explicitly designed to be stackable so that if one
autoloader cannot load a class another has a chance to do so. Having an autoloader
trigger a breaking error condition violates that compatibility.

2. `class_exists()` and `interface_exists()` allow "not found, even after trying to
autoload" as a legitimate, normal use case. An autoloader that throws exceptions
renders `class_exists()` unusable, which is entirely unacceptable from an interoperability
standpoint.  Autoloaders that wish to provide additional debugging information
in a class-not-found case should do so via logging instead, either to a PSR-3
compatible logger or otherwise.

Pros:

- Shallower directory structures

- More flexible file locations

- Stops underscore in class name from being honored as directory separator

- Makes implementations more explicitly interoperable

Cons:

- It is no longer possible, as under PSR-0, to merely examine a class name to
  determine where it is in the file system (the "class-to-file" convention
  inherited from Horde/PEAR).

### 4.2 Alternative: Stay With PSR-0 Only

Staying with PSR-0 only, although reasonable, does leave us with relatively
deeper directory structures.

Pros:

- No need to change anyone's habits or implementations

Cons:

- Leaves us with deeper directory structures

- Leaves us with underscores in the class name being honored as directory
  separators

### 4.3 Alternative: Split Up Autoloading And Transformation

Beau Simensen and others suggested that the transformation algorithm might be
split out from the autoloading proposal so that the transformation rules
could be referenced by other proposals. After doing the work to separate them,
followed by a poll and some discussion, the combined version (i.e.,
transformation rules embedded in the autoloader proposal) was revealed as the
preference.

Pros:

- Transformation rules could be referenced separately by other proposals

Cons:

- Not in line with the wishes of poll respondents and some collaborators

### 4.4 Alternative: Use More Imperative And Narrative Language

After the second vote was pulled by a Sponsor after hearing from multiple +1
voters that they supported the idea but did not agree with (or understand) the
wording of the proposal, there was a period during which the voted-on proposal
was expanded with greater narrative and somewhat more imperative language. This
approach was decried by a vocal minority of participants. After some time, Beau
Simensen started an experimental revision with an eye to PSR-0; the Editor and
Sponsors favored this more terse approach and shepherded the version now under
consideration, written by Paul M. Jones and contributed to by many.

### Compatibility Note with PHP 5.3.2 and below

PHP versions before 5.3.3 do not strip the leading namespace separator, so
the responsibility to look out for this falls on the implementation. Failing
to strip the leading namespace separator could lead to unexpected behavior.

## 5. People

### 5.1 Editor

- Paul M. Jones, Solar/Aura

### 5.2 Sponsors

- Phil Sturgeon, PyroCMS (Coordinator)
- Larry Garfield, Drupal

### 5.3 Contributors

- Andreas Hennings
- Bernhard Schussek
- Beau Simensen
- Donald Gilbert
- Mike van Riel
- Paul Dragoonis
- Too many others to name and count

## 6. Votes

- **Entrance Vote:** <https://groups.google.com/d/msg/php-fig/_LYBgfcEoFE/ZwFTvVTIl4AJ>

- **Acceptance Vote:**

    - 1st attempt: <https://groups.google.com/forum/#!topic/php-fig/Ua46E344_Ls>,
      presented prior to new workflow; aborted due to accidental proposal modification

    - 2nd attempt: <https://groups.google.com/forum/#!topic/php-fig/NWfyAeF7Psk>,
      cancelled at the discretion of the sponsor <https://groups.google.com/forum/#!topic/php-fig/t4mW2TQF7iE>

    - 3rd attempt: TBD

## 7. Relevant Links

- [Autoloader, round 4](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/lpmJcmkNYjM)
- [POLL: Autoloader: Split or Combined?](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/fGwA6XHlYhI)
- [PSR-X autoloader spec: Loopholes, ambiguities](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/kUbzJAbHxmg)
- [Autoloader: Combine Proposals?](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/422dFBGs1Yc)
- [Package-Oriented Autoloader, Round 2](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/Y4xc71Q3YEQ)
- [Autoloader: looking again at namespace](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/bnoiTxE8L28)
- [DISCUSSION: Package-Oriented Autoloader - vote against](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/SJTL1ec46II)
- [VOTE: Package-Oriented Autoloader](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/Ua46E344_Ls)
- [Proposal: Package-Oriented Autoloader](https://groups.google.com/forum/#!topicsearchin/php-fig/autoload/php-fig/qT7mEy0RIuI)
- [Towards a Package Oriented Autoloader](https://groups.google.com/forum/#!searchin/php-fig/package$20oriented$20autoloader/php-fig/JdR-g8ZxKa8/jJr80ard-ekJ)
- [List of Alternative PSR-4 Proposals](https://groups.google.com/forum/#!topic/php-fig/oXr-2TU1lQY)
- [Summary of [post-Acceptance Vote pull] PSR-4 discussions](https://groups.google.com/forum/#!searchin/php-fig/psr-4$20summary/php-fig/bSTwUX58NhE/YPcFgBjwvpEJ)



<h1 id="Example Implementations">Example Implementations of PSR-4</h1>

The following examples illustrate PSR-4 compliant code:

Closure Example
---------------

~~~php
<?php
/**
 * An example of a project-specific implementation.
 *
 * After registering this autoload function with SPL, the following line
 * would cause the function to attempt to load the \Foo\Bar\Baz\Qux class
 * from /path/to/project/src/Baz/Qux.php:
 *
 *      new \Foo\Bar\Baz\Qux;
 *
 * @param string $class The fully-qualified class name.
 * @return void
 */
spl_autoload_register(function ($class) {

    // project-specific namespace prefix
    $prefix = 'Foo\\Bar\\';

    // base directory for the namespace prefix
    $base_dir = __DIR__ . '/src/';

    // does the class use the namespace prefix?
    $len = strlen($prefix);
    if (strncmp($prefix, $class, $len) !== 0) {
        // no, move to the next registered autoloader
        return;
    }

    // get the relative class name
    $relative_class = substr($class, $len);

    // replace the namespace prefix with the base directory, replace namespace
    // separators with directory separators in the relative class name, append
    // with .php
    $file = $base_dir . str_replace('\\', '/', $relative_class) . '.php';

    // if the file exists, require it
    if (file_exists($file)) {
        require $file;
    }
});
~~~

Class Example
-------------

The following is an example class implementation to handle multiple
namespaces:

~~~php
<?php
namespace Example;

/**
 * An example of a general-purpose implementation that includes the optional
 * functionality of allowing multiple base directories for a single namespace
 * prefix.
 *
 * Given a foo-bar package of classes in the file system at the following
 * paths ...
 *
 *     /path/to/packages/foo-bar/
 *         src/
 *             Baz.php             # Foo\Bar\Baz
 *             Qux/
 *                 Quux.php        # Foo\Bar\Qux\Quux
 *         tests/
 *             BazTest.php         # Foo\Bar\BazTest
 *             Qux/
 *                 QuuxTest.php    # Foo\Bar\Qux\QuuxTest
 *
 * ... add the path to the class files for the \Foo\Bar\ namespace prefix
 * as follows:
 *
 *      <?php
 *      // instantiate the loader
 *      $loader = new \Example\Psr4AutoloaderClass;
 *
 *      // register the autoloader
 *      $loader->register();
 *
 *      // register the base directories for the namespace prefix
 *      $loader->addNamespace('Foo\Bar', '/path/to/packages/foo-bar/src');
 *      $loader->addNamespace('Foo\Bar', '/path/to/packages/foo-bar/tests');
 *
 * The following line would cause the autoloader to attempt to load the
 * \Foo\Bar\Qux\Quux class from /path/to/packages/foo-bar/src/Qux/Quux.php:
 *
 *      <?php
 *      new \Foo\Bar\Qux\Quux;
 *
 * The following line would cause the autoloader to attempt to load the
 * \Foo\Bar\Qux\QuuxTest class from /path/to/packages/foo-bar/tests/Qux/QuuxTest.php:
 *
 *      <?php
 *      new \Foo\Bar\Qux\QuuxTest;
 */
class Psr4AutoloaderClass
{
    /**
     * An associative array where the key is a namespace prefix and the value
     * is an array of base directories for classes in that namespace.
     *
     * @var array
     */
    protected $prefixes = array();

    /**
     * Register loader with SPL autoloader stack.
     *
     * @return void
     */
    public function register()
    {
        spl_autoload_register(array($this, 'loadClass'));
    }

    /**
     * Adds a base directory for a namespace prefix.
     *
     * @param string $prefix The namespace prefix.
     * @param string $base_dir A base directory for class files in the
     * namespace.
     * @param bool $prepend If true, prepend the base directory to the stack
     * instead of appending it; this causes it to be searched first rather
     * than last.
     * @return void
     */
    public function addNamespace($prefix, $base_dir, $prepend = false)
    {
        // normalize namespace prefix
        $prefix = trim($prefix, '\\') . '\\';

        // normalize the base directory with a trailing separator
        $base_dir = rtrim($base_dir, DIRECTORY_SEPARATOR) . '/';

        // initialize the namespace prefix array
        if (isset($this->prefixes[$prefix]) === false) {
            $this->prefixes[$prefix] = array();
        }

        // retain the base directory for the namespace prefix
        if ($prepend) {
            array_unshift($this->prefixes[$prefix], $base_dir);
        } else {
            array_push($this->prefixes[$prefix], $base_dir);
        }
    }

    /**
     * Loads the class file for a given class name.
     *
     * @param string $class The fully-qualified class name.
     * @return mixed The mapped file name on success, or boolean false on
     * failure.
     */
    public function loadClass($class)
    {
        // the current namespace prefix
        $prefix = $class;

        // work backwards through the namespace names of the fully-qualified
        // class name to find a mapped file name
        while (false !== $pos = strrpos($prefix, '\\')) {

            // retain the trailing namespace separator in the prefix
            $prefix = substr($class, 0, $pos + 1);

            // the rest is the relative class name
            $relative_class = substr($class, $pos + 1);

            // try to load a mapped file for the prefix and relative class
            $mapped_file = $this->loadMappedFile($prefix, $relative_class);
            if ($mapped_file) {
                return $mapped_file;
            }

            // remove the trailing namespace separator for the next iteration
            // of strrpos()
            $prefix = rtrim($prefix, '\\');
        }

        // never found a mapped file
        return false;
    }

    /**
     * Load the mapped file for a namespace prefix and relative class.
     *
     * @param string $prefix The namespace prefix.
     * @param string $relative_class The relative class name.
     * @return mixed Boolean false if no mapped file can be loaded, or the
     * name of the mapped file that was loaded.
     */
    protected function loadMappedFile($prefix, $relative_class)
    {
        // are there any base directories for this namespace prefix?
        if (isset($this->prefixes[$prefix]) === false) {
            return false;
        }

        // look through base directories for this namespace prefix
        foreach ($this->prefixes[$prefix] as $base_dir) {

            // replace the namespace prefix with the base directory,
            // replace namespace separators with directory separators
            // in the relative class name, append with .php
            $file = $base_dir
                  . str_replace('\\', '/', $relative_class)
                  . '.php';

            // if the mapped file exists, require it
            if ($this->requireFile($file)) {
                // yes, we're done
                return $file;
            }
        }

        // never found it
        return false;
    }

    /**
     * If a file exists, require it from the file system.
     *
     * @param string $file The file to require.
     * @return bool True if the file exists, false if not.
     */
    protected function requireFile($file)
    {
        if (file_exists($file)) {
            require $file;
            return true;
        }
        return false;
    }
}
~~~

### Unit Tests

The following example is one way of unit testing the above class loader:

~~~php
<?php
namespace Example\Tests;

class MockPsr4AutoloaderClass extends Psr4AutoloaderClass
{
    protected $files = array();

    public function setFiles(array $files)
    {
        $this->files = $files;
    }

    protected function requireFile($file)
    {
        return in_array($file, $this->files);
    }
}

class Psr4AutoloaderClassTest extends \PHPUnit_Framework_TestCase
{
    protected $loader;

    protected function setUp()
    {
        $this->loader = new MockPsr4AutoloaderClass;

        $this->loader->setFiles(array(
            '/vendor/foo.bar/src/ClassName.php',
            '/vendor/foo.bar/src/DoomClassName.php',
            '/vendor/foo.bar/tests/ClassNameTest.php',
            '/vendor/foo.bardoom/src/ClassName.php',
            '/vendor/foo.bar.baz.dib/src/ClassName.php',
            '/vendor/foo.bar.baz.dib.zim.gir/src/ClassName.php',
        ));

        $this->loader->addNamespace(
            'Foo\Bar',
            '/vendor/foo.bar/src'
        );

        $this->loader->addNamespace(
            'Foo\Bar',
            '/vendor/foo.bar/tests'
        );

        $this->loader->addNamespace(
            'Foo\BarDoom',
            '/vendor/foo.bardoom/src'
        );

        $this->loader->addNamespace(
            'Foo\Bar\Baz\Dib',
            '/vendor/foo.bar.baz.dib/src'
        );

        $this->loader->addNamespace(
            'Foo\Bar\Baz\Dib\Zim\Gir',
            '/vendor/foo.bar.baz.dib.zim.gir/src'
        );
    }

    public function testExistingFile()
    {
        $actual = $this->loader->loadClass('Foo\Bar\ClassName');
        $expect = '/vendor/foo.bar/src/ClassName.php';
        $this->assertSame($expect, $actual);

        $actual = $this->loader->loadClass('Foo\Bar\ClassNameTest');
        $expect = '/vendor/foo.bar/tests/ClassNameTest.php';
        $this->assertSame($expect, $actual);
    }

    public function testMissingFile()
    {
        $actual = $this->loader->loadClass('No_Vendor\No_Package\NoClass');
        $this->assertFalse($actual);
    }

    public function testDeepFile()
    {
        $actual = $this->loader->loadClass('Foo\Bar\Baz\Dib\Zim\Gir\ClassName');
        $expect = '/vendor/foo.bar.baz.dib.zim.gir/src/ClassName.php';
        $this->assertSame($expect, $actual);
    }

    public function testConfusion()
    {
        $actual = $this->loader->loadClass('Foo\Bar\DoomClassName');
        $expect = '/vendor/foo.bar/src/DoomClassName.php';
        $this->assertSame($expect, $actual);

        $actual = $this->loader->loadClass('Foo\BarDoom\ClassName');
        $expect = '/vendor/foo.bardoom/src/ClassName.php';
        $this->assertSame($expect, $actual);
    }
}
~~~
