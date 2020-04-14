.. title: Ternary predicates authorization in Spring Security
.. date: 2016-09-01 23:35:00 +0300
.. tags: java, spring-framework, oop
.. category: java
.. type: text

Recently I posted [an article on Spring Security]({% post_url 2016-08-06-java-8-libraries-and-android-applications-using-maven %}) describing how to make `@PreAuthorize` expressions custom types aware.
Today I'm going to share some my codebase for ternary authorization with three possible values: `GRANT`, `ABSTAIN`, and `DENY`.

<!-- TEASER_END -->

Why these three?
Sometimes simple boolean values are not enought in some situations.
Let's say you might want your authorization rules pass the authorization decision to the next decision-maker.
Merely boolean value will not let you do it.
Here the three is:

* `GRANT` &mdash; authorize the operation finally;
* `ABSTAIN` &mdash; defer the authorization decision to the next decision maker;
* `DENY` &mdash; deny the operation finally.

You might want to use `java.lang.Boolean` and `null` instead of `ABSTAIN`, but I think than the enum improves readability significantly.

In order to make the custom enum values combinable, let's implement the their composition rules using simple `NOT`, `AND` and `OR` operations.
Let me put the whole listing here:

```java
enum AuthorizationResult {

	GRANT,
	ABSTAIN,
	DENY;

	public final AuthorizationResult not() {
		return not(this);
	}

	public final AuthorizationResult or(final AuthorizationResult that) {
		return or(() -> that);
	}

	public final AuthorizationResult or(final Supplier<AuthorizationResult> that) {
		return or(this, that);
	}

	public final AuthorizationResult and(final AuthorizationResult that) {
		return and(() -> that);
	}

	public final AuthorizationResult and(final Supplier<AuthorizationResult> that) {
		return and(this, that);
	}

	static AuthorizationResult of(final boolean isSucceeded) {
		return isSucceeded ? GRANT : DENY;
	}

	static AuthorizationResult not(final AuthorizationResult op) {
		switch ( op ) {
		case GRANT:
			return DENY;
		case ABSTAIN:
			return ABSTAIN;
		case DENY:
			return GRANT;
		default:
			throw new AssertionError(op);
		}
	}

	static AuthorizationResult or(final AuthorizationResult op1, final Supplier<AuthorizationResult> op2Supplier) {
		switch ( op1 ) {
		case GRANT:
			return GRANT;
		case ABSTAIN:
			final AuthorizationResult op2F = op2Supplier.get();
			switch ( op2F ) {
			case GRANT:
				return GRANT;
			case ABSTAIN:
			case DENY:
				return ABSTAIN;
			default:
				throw new AssertionError(op2F);
			}
		case DENY:
			final AuthorizationResult op2D = op2Supplier.get();
			switch ( op2D ) {
			case GRANT:
				return GRANT;
			case ABSTAIN:
				return ABSTAIN;
			case DENY:
				return DENY;
			default:
				throw new AssertionError(op2D);
			}
		default:
			throw new AssertionError(op1);
		}
	}

	static AuthorizationResult and(final AuthorizationResult op1, final Supplier<AuthorizationResult> op2Supplier) {
		switch ( op1 ) {
		case GRANT:
			final AuthorizationResult op2P = op2Supplier.get();
			switch ( op2P ) {
			case GRANT:
				return GRANT;
			case ABSTAIN:
				return ABSTAIN;
			case DENY:
				return DENY;
			default:
				throw new AssertionError(op2P);
			}
		case ABSTAIN:
			final AuthorizationResult op2F = op2Supplier.get();
			switch ( op2F ) {
			case GRANT:
			case ABSTAIN:
				return ABSTAIN;
			case DENY:
				return DENY;
			default:
				throw new AssertionError(op2F);
			}
		case DENY:
			return DENY;
		default:
			throw new AssertionError();
		}
	}

}
```

See more at [three-valued logic at Wikipedia](https://en.wikipedia.org/wiki/Three-valued_logic).
Note the heavy use of `switch`.
I really love using enum and `switch` keyword as these two make a nice tandem.

One virtue of enum in `switch` is that switch may easily cover **all** enum constants (but not always can, because the total size of methods bytecode in Java is 64K).
In theory it can covert all bytes (in sense of `byte`) as well since there are 256 possible values, but I don't think it's a good idea of doing it using `switch`. :)

Another good thing is that we can be sure that a particular enum covers all enum values.
For example, IntelliJ IDEA can inspect for `switch`es missing enum cases, thus making the code more robust.

I also prefer the `default` case even if I put all enum constants to a `switch` &mdash; I really think this is good.
If an enum constant is accidentally missed, then the `default` case would process a missing enum value.
In really most cases it's better than doing nothing.
Also I usually terminate the `default` case with an `AssertionError` exception throwing, and I think it's good too.
It literally means "this must never happen" (I wouldn't like to have `ThisMustNeverHappenError` (not an `Exception`!) as `AssertionError` is semantically almost the same).
The "must never happen" also lets the compiler to detect the method termination paths well.

Now let's define an expression that could be handled in `@PreAuthorize`:

* `authorize` &mdash; must return the evaluation result;
* `compile` &mdash; must convert an expression to a human-readable form (this was one of the ideas of the previous post)
* `with` &mdash; a convenient static factory method to adapt both *something-to-authorization-result* function and a string consumer objects to an authorization predicate
* `not`, `and`, and `or` &mdash; all self-descriptive.

```java
interface IAuthorizationPredicate<T> {

	@Nonnull
	AuthorizationResult authorize(T t);

	void compile(@Nonnull final Consumer<String> consumer);

	static <T> IAuthorizationPredicate<T> with(final Function<? super T, AuthorizationResult> mapper, final Consumer<? super Consumer<String>> generator) {
		return new IAuthorizationPredicate<T>() {
			@Nonnull
			@Override
			public AuthorizationResult authorize(final T value) {
				return mapper.apply(value);
			}

			@Override
			public void compile(@Nonnull final Consumer<String> consumer) {
				generator.accept(consumer);
			}
		};
	}

	static <T> IAuthorizationPredicate<T> not(final IAuthorizationPredicate<? super T> predicate) {
		return with(
				t -> predicate.authorize(t).not(),
				c -> {
					c.accept("(NOT ");
					predicate.compile(c);
					c.accept(")");
				}
		);
	}

	default IAuthorizationPredicate<T> or(final IAuthorizationPredicate<? super T> other) {
		return with(
				t -> authorize(t).or(() -> other.authorize(t)),
				c -> {
					c.accept("(");
					compile(c);
					c.accept(" OR ");
					other.compile(c);
					c.accept(")");
				}
		);
	}

	default IAuthorizationPredicate<T> and(final IAuthorizationPredicate<? super T> other) {
		return with(
				t -> authorize(t).and(() -> other.authorize(t)),
				c -> {
					c.accept("(");
					compile(c);
					c.accept(" AND ");
					other.compile(c);
					c.accept(")");
				}
		);
	}

}
```

This lets us to create a custom set of already-defined predicates.
Let's say:

```java
private static final IAuthorizationPredicate<Administrator> anybody = with(
		a -> of(a != null),
		c -> c.accept("anybody")
);

private static final IAuthorizationPredicate<Administrator> root = with(
		a -> of(a.isRoot()),
		c -> c.accept("root")
);

private static final IAuthorizationPredicate<Administrator> inRootGroup = with(
		a -> of(a.getAdministratorGroups().stream().anyMatch(AdministratorGroup::isRoot)),
		c -> c.accept("inRootGroup")
);

static IAuthorizationPredicate<Administrator> me(final long administratorId) {
	return with(
			a -> of(a.getId() == administratorId),
			c -> c.accept("me")
	);
}

static IAuthorizationPredicate<Administrator> isRootId(final long administratorId) {
	return with(
			a -> of(isRootAdministrator(administratorId)),
			c -> c.accept("isRootId")
	);
}

private static final IAuthorizationPredicate<Administrator> hasAdministratorRoleEnabled = with(
		a -> of(a.getAdministratorGroups().stream().anyMatch(ag -> ag.isAdministrator() &amp;&amp; ag.isEnabled())),
		c -> c.accept("hasAdministratorRoleEnabled")
);
```

All of these fields and methods (static getters omitted by intention) define special *named* predicates that define different authorization rules.
And now, in an authorization component that might be used in a `@PreAuthorize` expression:

```java
me(administratorId)
		.or(root())
		.or(hasAdministratorRoleEnabled().and(not(isRootId(administratorId))))
```

I hope it can be easily read.
Let's read loud:

```
Authorize the given principal
WHEN
the administator ID is the ID of mine
OR
the given principal is the root
OR
the given principal group has administrator role enable AND the given administrator ID is NOT root
```

It formally means *may update anybody except of the root if and only if the administrator ID refers to a user with administrative permissions*.
The `AuthorizationResult` can be converted to `boolean` as it was told in the linked post, and your authorization components might return `AuthorizationResult` in `@PreAuthorize` expressions.
It just makes the code cleaner.
