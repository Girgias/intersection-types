# PHP RFC: Pure intersection types

  * Version: 0.1
  * Date: 2021-03-23
  * Author: George Peter Banyard, <girgias@php.net>
  * Status: Draft
  * Target Version: PHP 8.1
  * Implementation: https://github.com/php/php-src/pull/6799
  * First Published at: http://wiki.php.net/rfc/pure-intersection-types
  * GitHub mirror: https://github.com/Girgias/intersection-types

## Introduction

An "intersection type" requires a value to satisfy multiple type constraints instead of a single one.

Intersection types are currently not supported natively by the language. Instead, one must either use phpdoc annotations, and/or abuse typed properties [1] as can be seen in the following example:

```php
class Test {
	private ?Traversable $traversable = null;
	private ?Countable $countable = null;
	/** @var Traversable&Countable */
	private $both = null;
	
	public function __construct($countableIterator) {
		$this->traversable =& $this->both;
		$this->countable =& $this->both;
		$this->both = $countableIterator;
	}
}
```

Supporting intersection types in the language allows us to move more type information from phpdoc into function signatures, with the usual advantages this brings:

  * Types are actually enforced, so mistakes can be caught early.
  * Because they are enforced, type information is less likely to become outdated or miss edge-cases.
  * Types are checked during inheritance, enforcing the Liskov Substitution Principle.
  * Types are available through Reflection.
  * The syntax is a lot less boilerplate-y than phpdoc.

## Proposal

Add support for pure intersection types are specified using the syntax `T1&T2&...` and can be used in all positions where types are currently accepted:

```php
class A {
    private Traversable&Countable $countableIterator;

    public function setIterator(Traversable&Countable $countableIterator): void {
        $this->countableIterator = $countableIterator;
    }

    public function getIterator(): Traversable&Countable {
        return $this->countableIterator;
    }
}
```

This means it would *not* be possible to mix intersection and union types together such as `A&B|C`,
this is left as a future scope.

### Supported types

Only class types (interfaces and class names) are supported by intersection types.

The rationale is that for nearly all standard types using them in an intersection type result
in a type which can never be satisfied (e.g. `int&string`).

Usage of `mixed` in an intersection type is redundant as `mixed&T` corresponds to `T`, as such this is disallowed.

Similarly using `iterable` in an intersection results in a redundant invalid type, this can be seen by expanding
the type expression `iterable&T = (array|Traversable)&T = (array&T) | (Traversable&T) =  Traversable&T`

Although an intersection with `callable` *can* make sense (e.g. string&callable),
we think it is unwise and points to a bug.

Using `self`, `static`, or `parent` is also forbidden in an intersection type as these correspond to concrete classes.

#### Duplicate and redundant types

To catch some simple bugs in intersection type declarations, redundant types that can be detected without performing class loading will result in a compile-time error. This includes:

  * Each name-resolved type may only occur once. Types like `A&B&A` result in an error.

This does not guarantee that the type is "minimal", because doing so would require loading all used class types.

For example, if `A` and `B` are runtime class aliases, then `A&B` remains a legal intersection type, even though it could be reduced to either `A` or `B`. Similarly, if `class B extends A {}`, then `A&B` is also a legal intersection type, even though it could be reduced to just `B`.

```php
function foo(): A&A {} // Disallowed

use A as B;
function foo(): A&B {} // Disallowed ("use" is part of name resolution)

class_alias('X', 'Y');
function foo(): X&Y {} // Allowed (redundancy is only known at runtime)
```


#### Type grammar

Due to a parser ambiguity with the declaration of by-ref parameter while using the current LR(1) parser,
the grammar and lexer are modified to create different tokens for the `&` character depending if it is
followed by a (variadic) variable or not.

The grammar thus looks as following:

```
type_expr:
		type	
	|	'?' type	
	|	union_type
	|	intersection_type
;


intersection_type:
		type T_AMPERSAND_NOT_FOLLOWED_BY_VAR_OR_VARARG type
	|	intersection_type T_AMPERSAND_NOT_FOLLOWED_BY_VAR_OR_VARARG type
;
```

### Variance

Intersection types follow standard PHP variance rules that are already used for inheritance and type checking:

  * Return types are covariant (child must be subtype).
  * Parameter types are contravariant (child must be supertype).
  * Property types are invariant (child must be subtype and supertype).

The only change is in how intersection types interact with subtyping, with two additional rules:

  * `A` is a subtype of `B_1&...&B_n` if for all `B_i`, `A` is a subtype of `B_i`
  * `A_1&...&A_n` is a subtype of `B` if there exists an `A_i` such that `A_i` is a subtype of `B`

In the following, some examples of what is allowed and what isn't are given.

#### Property types

Property types are invariant, which means that types must stay the same during inheritance. However, the "same" type may be expressed in different ways.

Intersection types expand the possibilities in this area: For example `A&B` and `B&A` represent the same type. The following example shows a more complex case:

```php
class A {}
class B extends A {}

class Test {
    public A&B $prop;
}
class Test2 extends Test {
    public B $prop;
}
```

In this example, the intersection `A&B` actually represents the same type as just `B`, and this inheritance is legal, despite the type not being syntactically the same.

Formally, we arrive at this result as follows: First, the parent type `A&B` is a subtype of `B`. Second, `B` is a subtype of `A&B`, because `B` is a subtype of `A` and `B` is a subtype of `B`.

#### Adding and removing intersection types

It is legal to add intersection types in return position and remove intersection types in parameter position:

```php
class A {}
interface X {}

class Test {
    public function param1(A $param) {}
    public function param2(A&X $param) {}

    public function return1(): A&X {}
    public function return2(): A {}
}

class Test2 extends Test {
    public function param1(A&X $param) {}   // FORBIDDEN: Adding extra param type constraint
    public function param2(A $param) {}     // Allowed: Removing param type constraint

    public function return1(): A {}         // FORBIDDEN: Removing return type constraint
    public function return2(): A&X {}       // Allowed: Adding extra return type constraint
}
```

#### Variance of individual intersection members

Similarly, it is possible to restrict an intersection member in return position,
or widen an intersection member in parameter position:

```php
class A {}
class B extends A {}
interface X {}

class Test {
    public function param1(B&X $param) {}
    public function param2(A&X $param) {}

    public function return1(): A&X {}
    public function return2(): B&X {}
}

class Test2 extends Test {
    public function param1(A&X $param) {} // Allowed: Widening intersection member B -> A
    public function param2(B&X $param) {} // FORBIDDEN: Restricting intersection member A -> B

    public function return1(): B&X {}     // Allowed: Restricting intersection member A -> B
    public function return2(): A&X {}     // FORBIDDEN: Widening intersection member B -> A
}
```

Of course, the same can also be done with multiple intersection members at a time, and be combined with the addition/removal of types mentioned previously.

#### Variance of intersection type to concrete class type

As the primary use of intersection types is to ensure multiple interfaces are implemented,
a concrete class or interface which implements all the interfaces present in the intersection
is considered a subtype and thus can be used where co-variance is allowed.

```php
interface X {}
interface Y {}

class TestOne implements X, Y {}

interface A
{
    public function foo(): X&Y;
}


interface B extends A
{
    public function foo(): TestOne;
}
```

Moreover, it is possible to use a union type of concrete classes/interface when each of the
member of the union implement all of the interfaces in the intersection.


```php
class TestTwo implements X, Y {}

interface C extends A
{
    public function foo(X&Y $param): TestOne|TestTwo;
}
```

The reason why this is possible is that a union of concrete classes/interfaces is less
general then the set of possible classes which satisfy the intersection type.


### Coercive typing mode

As standard types are not allowed in pure intersection types,
no consideration for the coercive typing mode needs to done.


### Property types and references

References to typed properties with intersection types follow the semantics outlined in the [typed properties RFC](https://wiki.php.net/rfc/typed_properties_v2#general_semantics):

> If typed properties are part of the reference set, then the value is checked against each property type. If a type check fails, a TypeError is generated and the value of the reference remains unchanged.

```php

interface X {}
interface Y {}
interface Z {}

class A implements X, Y, Z {}
class B implements X, Y {}

class Test {
    public X&Y $y;
    public X&Z $z;
}
$test = new Test;
$r = new A;
$test->y =& $r;
$test->z =& $r;

// Reference set: { $r, $test->y, $test->z }
// Types: { A, X&Y, X&Z }

$r = new B;
// TypeError: Cannot assign B to reference held by property Test::$z of type X&Z
```

### Reflection

To support intersection types, a new class `ReflectionIntersectionType` is added:

```php
class ReflectionIntersectionType extends ReflectionType {
    /** @return ReflectionType[] */
    public function getTypes();

    /* Inherited from ReflectionType */
    /** @return bool */
    public function allowsNull();

    /* Inherited from ReflectionType */
    /** @return string */
    public function __toString();
}
```

The `getTypes()` method returns an array of `ReflectionType`s that are part of the intersection. The types may be returned in an arbitrary order that does not match the original type declaration. The types may also be subject to equivalence transformations.

For example, the type `X&Y` may return types in the order `["Y", "X"]` instead.
The only requirement on the Reflection API is that the ultimately represented type is equivalent.

The `__toString()` method returns a string representation of the type that constitutes a valid code representation of the type in a non-namespaced context. It is not necessarily the same as what was used in the original code.

### Examples

```php
// This is one possible output, getTypes() and __toString() could
// also provide the types in the reverse order instead.
function test(): A&B {}
$rt = (new ReflectionFunction('test'))->getReturnType();
var_dump(get_class($rt));    // "ReflectionIntersectionType"
var_dump($rt->allowsNull()); // false
var_dump($rt->getTypes());   // [ReflectionType("A"), ReflectionType("B")]
var_dump((string) $rt);      // "A&B"

function test2(): A&B&C {}
$rt = (new ReflectionFunction('test2'))->getReturnType();
var_dump(get_class($rt));    // "ReflectionIntersectionType"
var_dump($rt->allowsNull()); // false
var_dump($rt->getTypes());   // [ReflectionType("A"), ReflectionType("B"),
                             //  ReflectionType("C")]
var_dump((string) $rt); // "A&B&C"
```


## Backward Incompatible Changes

This RFC does not contain any backwards incompatible changes.

However, existing `ReflectionType` based code might need to be adjusted in order to support processing of code that uses intersection types.

## Proposed PHP Version

Next minor version, i.e. PHP 8.1.

## Future Scope

The features discussed in the following are **not** part of this proposal.

### Composite types (i.e. mixing union and intersection types)

While early prototyping [2] shows that supporting `A&B|C` without any grouping looks feasible,
there are still many other considerations (e.g. Reflection), but namely the variance rules and checks,
which would be dramatically increased and prone to error.

There is also the opinion that composite types should not rely on precedence of unions but be explicitly grouped together.

As such we consider a stepped approach by only allowing pure intersection first the best way forward.

### Type Aliases

As types become increasingly complex, it may be worthwhile to allow reusing type declarations. There are two general ways in which this could work. One is a local alias, such as:

```
use Traversable&Countable as CountableIterator;
 
function foo(CountableIterator $x) {}
```

In this case `CountableIterator` is a symbol that is only visible locally and will be resolved to the original `Traversable&Countable` type during compilation.

The second possibility is an exported typedef:

```
namespace Foo;
type CountableIterator = Traversable&Countable;
 
// Usable as \Foo\CountableIterator from elsewhere
```

It should be noted that inclusion of this proposal will add extra considerations for type aliases as it would be possible to
write composite types as if grouping was supported. However, the groundwork for supporting this is present in this proposal.

## Proposed Voting Choices

As per the voting RFC a yes/no vote with a 2/3 majority is needed for this proposal to be accepted.

## Patches and Tests 
Links to any external patches and tests go here.

If there is no patch, make it clear who will create a patch, or whether a volunteer to help with implementation is needed.

Make it clear if the patch is intended to be the final patch, or is just a prototype.

For changes affecting the core language, you should also provide a patch for the language specification.

## Implementation 
After the project is implemented, this section should contain
  - the version(s) it was merged into
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature
  - a link to the language specification section (if any)
  
## Acknowledgements

To Ilija Tovilo for resolving the parser conflict with by-ref parameters.

## References

[1]: Slide 14 of Nikita Popov's talk "Typed Properties and more: What's coming in PHP 7.4?" <https://image.slidesharecdn.com/presentationnikita-190519190251/95/typed-properties-and-more-whats-coming-in-php-74-14-638.jpg?cb=1558292620>  
[2]: Git PR with basic prototype for mixing intersection and union types <https://github.com/Girgias/php-src/pull/8>  
