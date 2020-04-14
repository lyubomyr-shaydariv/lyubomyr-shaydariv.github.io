.. title: Optional<T> tends to be used in a wrong way
.. date: 2017-01-26 03:12:00 +0300
.. tags: java, java-8
.. category: java
.. type: text

Want to explicitly declare a nullability contract?
You probably might want to use `Optional<T>`, but it does not seem to be the best way.
I've got some cons against it.

<!-- TEASER_END -->

## The cons

### It's just another object created to enclose a nullable value

Hit the heap?

### `Iterable`s of `Optional`s are more expensive than `Iterable`s of "regular" objects

This one is derived from the latter point.

### It still requires checking for `isPresent`

... and thus bringing nullability checks to runtime, otherwise it fails with `NoSuchElementException`.
How much does it differ from `NullPointerException`?
Is it disguisted?

### Standard functional interfaces like `Consumer`, `Supplier`, and `Function` in Java 8 are not checked-exceptions friendly

Sad truth for those who prefer `ifPresent`.

### `Optional<T>` may lack support by (de)serialization libraries

Unless they're updated either by you or by the maintainers of the libraries.
Moreover, `Optional` is not `Serializable`.
Have fun.

### `Optional.class` can't say anything of the parameterization

The Google Guava and Google Gson `TypeToken` and the Spring Framework `ParameterizedTypeReference` _can_ help in some use cases.
Some frameworks and libraries can only spot fields by classes or primitites types, but not generics what `Optional<T>` is.

### What if use `emptyList()`/`singletonList(T)` as a surrogate `Optional<T>`?

Seriously, why not?
Semantically these two really does not differ much from `Optional` however having another methods exposed.
`!isEmpty` just stands for `isPresent`.

### `Optional<T>` only means _there can be no value_

It's a clear implementation of the [Null Object pattern](https://en.wikipedia.org/wiki/Null_Object_pattern).
Null objects _are not_ `null`s and have _some_ default _null object_ behavior whilst `null` cannot have any behavior.

### `Optional<T>` can be `null` itself and you can't ever avoid it

Nuff said.

## What's then?

Use [JSR-305](https://jcp.org/en/jsr/detail?id=305) `@Nullable` and `@Nonnull` wisely.

### Interface methods return values

Bad:

```java
public interface IService {

	String getId();

	String getName();

}
```

Good:

```java
public interface IService {

	@Nonnull
	String getId();

	@Nullable
	String getName();

}
```

The interface method return value types are more aligned now, right?

### Another case

Bad:

```java
public static Optional<String> foo() {
	...
}
```

Good:

```java
@Nullable
public static String foo() {
	...
}
```

Now you can do whatever you want: either check it for `null` or wrap it in an `Optional.ofNullable` _if you really like it_.
Don't let the code force you to follow excessive "conventions".

### Class fields

Bad:

```java
public final class Box<T> {

	private final Optional<T> value;

	...

	public Optional<T> getValue() {
		return value;
	}

}
```

Good:

```java
public final class Box<T> {

	@Nullable
	private final T value;

	...

	@Nullable
	public T getValue() {
		return value;
	}

}
```

Just no fields bloating.

### Method arguments

Bad:

```java
public static void foo(final Optional<String> value) {
	...
}
```

Good:

```java
public static void foo(@Nullable final String value) {
	...
}
```

Yes, I know, some `Optional`-oriented folks may say they don't use `Optional` as method parameters.
Why not aligning to returned values?
Another point for the latter piece of code is that the `foo` method does not force you to pass an `Optional` to it, and it can wrap it up when it needs it on its own.

## As the very end

And don't trust [Uses for Optional](http://stackoverflow.com/questions/23454952/uses-for-optional) much.

You might also like to read a more detailed article [Nothing is better than the Optional type](https://homes.cs.washington.edu/~mernst/advice/nothing-is-better-than-optional.html) by Michael Ernst.
