# Custom equals in inline classes

* **Type**: Design proposal
* **Authors**: Vladislav Grechko
* **Status**: Prototype implementation in progress
* **Issue**: [KT-24874](https://youtrack.jetbrains.com/issue/KT-24874/)

## Summary

Allow overriding `equals` from `Any` and defining so-called 'typed equals' in inline classes.

## Motivation and use cases

### Collections

Suppose we would like to implement wrapper for an object that implements `List` interface. We want this wrapper to be
light-weight and avoid allocation of wrapper object.
This might be achieved by declaring an inline class:

```Kotlin
@JvmInline
value class SingletonList<T>(val element: T) : List<T> {
    ...
}
```

> Note that passing instance of `SingletonList` as argument to non-inline function that takes `List`  will lead to
> boxing and allocation a new object. For inline functions, boxing can be avoided when IR inliner is implemented.
> Related issue:
> [KT-40391](https://youtrack.jetbrains.com/issue/KT-40391/Inline-function-boxes-inline-class-type-argument)
>
Since different implementations of `List` must be equal if they contain same elements, we have to
customize `equals(other: Any?)` in `SingletonList`.

### Units of measurement

Inline classes are well suited to represent units of measurement. It might be useful to establish equivalence relation
on instances of such classes:

```Kotlin
@JvmInline
value class Degrees(val value: Double)


fun foo() {
    // customize equality relation to make the set contain a single value
    val uniqueAngles = setOf(Degrees(90.0), Degrees(-270.0))
}
```

```Kotlin
@JvmInline
value class Minutes(val value: Int)

@JvmInline
value class Seconds(val value: Int)


fun foo() {
    // customize equality relation to make the set contain a single value
    val uniqueTimeIntervals = setOf(Minutes(1), Seconds(60))
}
```

It is obvious that the ability to override `equals` brings necessity of the ability to override `hashCode`.

## Proposal

### Typed equals

Simply overriding `equals(other: Any?)` in inline class would require boxing of the right-hand side operand of
every '==' expression:

```Kotlin
@JvmInline
value class Degrees(val value: Double) {
    override fun equals(other: Any?): Boolean {
        ...
    }
}

fun foo() {
    // have to box to pass to equals(other: Any?)
    println(Degrees(0) == Degrees(45))
}
```

That is why in [KEEP for inline classes](https://github.com/Kotlin/KEEP/blob/master/proposals/inline-classes.md) a new
concept of **typed equals** was proposed:

```Kotlin
@JvmInline
value class Degrees(val value: Double) {
    fun equals(other: Degrees) = (value - other.value) % 360.0 == 0.0
}
```

Type of `other` will be erased and passing argument to the functions will not require boxing.

More precise, we define typed equals as a function such that:

* Has name `"equals"`
* Declared as a member function of inline class
* Has a single parameter which type is a star-projection of enclosing class
    * We will elaborate on this restriction in the next section
* Returns `Boolean`
* Annotated with `@TypedEquals` annotation

Typed equals, as well as equals from `Any`, must define an equivalence relation, i.e. be symmetric, reflexive,
transitive and consistent.

We forbid typed equals to have type parameters.

> From now, we will be calling equals from `Any` *untyped* equals, in contrast, to *typed* one.

### Interaction of typed and untyped equals

We need to establish consistency of typed and untyped equals, to be more precise, the following property
> For any `x`, `y` such that `x`, `y` refer instances of an inline class, it must be true
> that `x.equals(y) == x.equals(y as Any?)`

We propose the following:

* In case only typed equals if defined, generate consisted untyped one. Implementing untyped equals will be trying to
  cast the argument to correspond inline class and passing to typed equals if successful.
* In case only untyped equals if defined, generated consisted typed one. Implementing typed equals will be boxing
  argument and passing it to untyped equals. Implement special diagnostics to warn programmers about boxing.
* In case both methods are defined, make the programmer responsible for their consistency.

Note that in the first case on JVM level we will be casting argument to the raw type of inline class, thus we will know
nothing about type arguments. We emphasize this fact by requiring the parameter of typed equals to be star-projection of
the inline class.

## Other questions

### Inline classes over inline classes

Suppose we have an inline class `B` over inline class `A` and `A` defines typed equals. In this case,
default-generated equals in `B` will be comparing it underlying values using custom equals of `A`.

### Interaction with Valhalla

In its current implementation, Valhalla supports customization of equals and hashCode in values classes, thus, we will
be able to compile Kotlin inline classes with custom equals to Valhalla values classes.

```Java
value class ValueClass {
    int x;

    public ValueClass(int x) {
        this.x = x;
    }

    @Override
    public boolean equals(Object other) {
        return true;
    }

    @Override
    public int hashCode() {
        return 42;
    }
}

class Main {
    public static void main(String[] args) {
        ValueClass zeroVal = new ValueClass(0);
        ValueClass oneVal = new ValueClass(1);

        Object zeroObj = zeroVal;

        System.out.println(zeroVal.equals(oneVal));        // true
        System.out.println(zeroObj.equals(oneVal));        // true
        System.out.println(zeroObj.hashCode());            // 42
    }
}
```

## Future improvements

### Resolving references to typed equals in compiler frontend

Currently `'=='` operator is being resolved to typed equals in compiler backend, but resolving it in frontend could help
user to navigate to definition of typed equals from `'=='` invocation.

### Typed equals in non-value classes

Introducing typed equals in non-value classes could help to get rid of boilerplate code:

```Kotlin
class A {
    fun equals(other: A) {
        //do check
        ...
    }
}
```

instead of:

```Kotlin
class A {
    override fun equals(other: Any?) {
        if (other is A) {
            //do check
            ...
        }
        return false
    }
}
```