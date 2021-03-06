# PHAN

**Phan** is a static analyzer for PHP.

It requires PHP 7 with the [php-ast][phpast] extension loaded. The code you
analyze with it can be written for any version of PHP, of course.

It is nowhere near ready for production use. If you need a production-quality
solution right now, you should probably look at [scrutinizer].

## Getting it working

If you already have PHP 7 somewhere, it should be trivial. If not, you could grab
my [php7dev][php7dev] Vagrant image or one of the many Docker builds out there.

Then compile [php-ast][phpast]. Something along these lines should do it:

```
git clone https://github.com/nikic/php-ast.git
cd php-ast
phpize
./configure
make install
```

And add `extension=ast.so` to your `php.ini` file. Check that it is there with `php -m`.
If it isn't you probably added it to the wrong `php.ini` file. Check `php --ini` to see
where it is looking.

## Usage

```
phan *.php
```

or give it a text file containing a list of files (but see the next section) to scan:

```
phan -f filelist.txt
```

and it might generate output that looks like this:

```
Files scanned:	14
Time:		0.21s
Classes:	7
Methods:	9
Functions:	20
Closures:	1
Traits:		0
Conditionals:	281
Issues found:	11

test1.php:191 UndefError call to undefined function get_real_size()
test1.php:232 UndefError static call to undeclared class core\session\manager
test1.php:386 UndefError Trying to instantiate undeclared class lang_installer
test2.php:4 TypeError arg#1(arg) is object but escapeshellarg() takes string
test2.php:4 TypeError arg#1(msg) is int but logmsg() takes string defined at sth.php:5
test2.php:4 TypeError arg#2(level) is string but logmsg() takes int defined at sth.php:5
test3.php:11 TypeError arg#1(number) is string but number_format() takes float
test3.php:12 TypeError arg#1(string) is int but htmlspecialchars() takes string
test3.php:13 TypeError arg#1(str) is int but md5() takes string
test3.php:14 TypeError arg#1(separator) is int but explode() takes string
test3.php:14 TypeError arg#2(str) is int but explode() takes string
```

To make sure it works you can run `phan` on itself with `phan -f filelist.txt` using
`filelist.txt` provided here. But a better way to check is to run `./run-tests.php`.

## Generating a file list

This static analyzer does not track includes or try to figure out autoloader magic. It treats
all the files you throw at it as one big application. For code encapsulated in classes this
works well. For code running in the global scope it gets a bit tricky because order
matters. If you have an `index.php` including a file that sets a bunch of global variables and
you then try to access those after the include in `index.php` the static analyzer won't
know anything about these.

In practical terms this simply means that you should put your entry points and any files
setting things in the global scope at the top of your file list. If you have a `config.php`
that sets global variables that everything else needs put that first in the list followed by your
various entry points, then all your library files containing your classes.

## Features

* Checks for calls and instantiations of undeclared functions, methods, closures and classes
* Checks types of all arguments and return values to/from functions, closures and methods
* Supports [phpdoc][doctypes] comments including union and void/null types
* Checks for [Uniform Variable Syntax][uniform] PHP 5 -> PHP 7 BC breaks
* Undefined variable tracking
* Supports namespaces, traits and variadics
* Generics (from phpdoc hints - int[], string[], UserObject[], etc.)
* Basic tainted data detection

See the [tests][tests] directory for some examples of the various checks.

## Planned

* [Obsolescence](#obsolescence)
* Array element tracking (generics-like)
* Class consts and properties
* JSON, csv and possibly other output formats
* Perhaps a genlist feature that will automatically figure out dependency files starting
  from a single entry point file

## Bugs

There are plenty of them. Especially related to assumptions made during variable tracking.

When you find one, please take the time to create a tiny reproducing code snippet that illustrates
the bug. And once you have done that, fix it. Then turn your code snippet into a test and add it to
[tests][tests] then `./run-tests.php` and send me a PR with your fix and test.

## How it works

One of the big changes in PHP 7 is the fact that the parser now uses a real
Abstract Syntax Tree ([AST][php7ast]). This makes it much easier to write code
analysis tools by pulling the tree and walking it looking for interesting things.

Phan has 2 passes. On the first pass it reads every file, gets the AST and recursively parses it
looking only for functions, methods and classes in order to populate a bunch of
global hashes which will hold all of them. It also loads up definitions for all internal
functions and classes. The type info for these come from a big file called [arginfo.php][arginfo].
[Pass1][pass1] is quite simple to follow.

The real complexity hits you hard in [Pass2][pass2]. Here some things are done recursively depth-first
and others not. For example, we catch something like `foreach($arr as $k=>$v)` because we need to tell the
foreach code block that `$k` and `$v` exist. For other things we need to recurse as deeply as possible
into the tree before unrolling our way back out. For example, for something like `c(b(a(1)))` we need
to call `a(1)` and check that `a()` actually takes an int, then get the return type and pass it to `b()`
and check that, before doing the same to `c()`.

There is a `$scope` global hash which keeps track of all variables. It mimics PHP's scope handling in that it
has a `$scope['global']` along with entries for each function, method and closure. This is used to detect
undefined variables and also type-checked on a `return $var`. There is a debugging feature which will let
you dump the scope. If `test.php` is:

```php
<?php
class MyClass {
	static function init(string $arg1, $arg2='opt'):array {
		$local = [$arg1, $arg2];
		return $local;
	}

	function read():string {
		return "abc";
	}
}

$stuff = MyClass::init("one");
$data  = $stuff->read();
```

then `phan -s test.php` will output:

```
MyClass::init
¯¯¯¯¯¯¯¯¯¯¯¯¯
 Variables:
	arg1: string (param: 1)
	arg2: string (param: 2)
	local: array

MyClass::read
¯¯¯¯¯¯¯¯¯¯¯¯¯
 Variables:
	this: object:MyClass

global
¯¯¯¯¯¯
 Variables:
	_GET: array(tainted)
	_POST: array(tainted)
	_COOKIE: array(tainted)
	_REQUEST: array(tainted)
	_SERVER: array(tainted)
	_FILES: array(tainted)
	GLOBALS: array
	stuff: array
	data: mixed
```
Note that undefined variable checking in the global scope will depend on the order you scan the files in.
So generally you will want to put any entry point files that do things in the global scope before all your
library files.

The `$classes` hash has a list of all classes, both internal and user-space. This is walked to figure out
inheritance on calls by checking `$classes['class_name']['parent']`.

The `$functions` hash contains all the known functions both internal and userspace. There is a debugging flag
that will let you dump user-defined classes and functions. For the previous example `test.php` if you run `phan -u test.php`
you will get:

```
class MyClass
¯¯¯¯¯¯¯¯¯¯¯¯¯
	 MyClass::init(string arg1, arg2):array
	 MyClass->read():string
```

The `$namespace_map` global contains the per-file namespacing aliasing.

The `$quick_mode` global is just a flag that tells Pass 2 whether or not to rescan every function, method
and closure call with the current set of args. By default quick mode is off, but you might want to turn it
on if you are trying to run this on a huge codebase as part of a Continuous Integration process. Quick mode
can be 2 to 3 times faster but it will miss a few potential issues as explained below.

## Quick Mode Explained

In Quick-mode the scanner doesn't rescan a function or a method's code block every time
a call is seen. This means that the problem here won't be detected:

```php
<?php
function test($arg):int {
	return $arg;
}
test("abc")
```

This would normally generate:

```
test.php:3 TypeError return string but `test()` is declared to return int
```

The initial scan of the function's code block has no type information for `$arg`. It
isn't until we see the call and rescan test()'s code block that we can detect
that it is actually returning the passed in `string` instead of an `int` as declared.

## How you can help

This is written in PHP specifically to encourage contributions. I know my stream-of-conciousness coding
style can be a bit hard to sort through. The bulk of this was written quickly over a weekend with
almost no re-factoring and cleanup yet. Please give me a hand with that.

Look through the code for `TODO` comments and see if you can tackle any of those.

And this last one is something anyone can help with. The [arginfo.php][arginfo] file is not
complete. It was generated and then hand-edited but with currently around 8500 entries, there
are mistakes. Many of them even. You will notice them when you scan your own code. Please help me
fix it. Hopefully the format is self-explanatory, especially if you read the comment at the top
of the file.

### Obsolescence

I don't actually want to write, nor maintain a static analyzer. This is a placeholder and
proof-of-concept designed to inspire and enourage others to write something better. We need
a practical and pragmatic analyzer that we can just point at a bunch of code and have it
tell us about any issues in it. Bits and pieces of this, especially [arginfo.php][arginfo]
and maybe the tests will likely surive long-term, but much of it probably won't.

  [phpast]: https://github.com/nikic/php-ast
  [scrutinizer]: https://scrutinizer-ci.com/docs/guides/php/automated-code-reviews
  [doctypes]: http://www.phpdoc.org/docs/latest/guides/types.html
  [tests]: https://github.com/rlerdorf/phan/blob/master/tests
  [php7ast]: https://wiki.php.net/rfc/abstract_syntax_tree
  [arginfo]: https://github.com/rlerdorf/phan/blob/master/includes/arginfo.php
  [pass1]: https://github.com/rlerdorf/phan/blob/master/includes/pass1.php
  [pass2]: https://github.com/rlerdorf/phan/blob/master/includes/pass2.php
  [mainloop]: https://github.com/rlerdorf/phan/blob/master/phan
  [php7dev]: https://github.com/rlerdorf/php7dev
  [uniform]: https://wiki.php.net/rfc/uniform_variable_syntax
