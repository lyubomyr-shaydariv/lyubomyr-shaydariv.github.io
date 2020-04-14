.. title: The Java language seems to go a wrong way
.. date: 2018-05-14 09:00:00 +0300
.. tags: java
.. category: java
.. type: text

I'm not sure if the Java language goes a right way.
The new Java language features look really not I would like to have in Java.

<!-- TEASER_END -->

When I started my professional career, I started it with C# 2.0 and C# 3.0.
These were beatufiul to me and I missed many features from them when I switched to Java back in 2010.
I didn't have an opportunity to work with C# 4.0 and later, but now C# looks like a monstrous language to me.
And it's sad to realize that the Java language gathers a lot of.

### Local-variable type inference

* It just decreases readability drastically. No need to say about a trivial example of right-hand `new ArrayList<...>()`, because it hurts to know what some other expression returns. If there is a method at the right-hand, why do I need to use an IDE just to remind that the method returns something cryptic else? `var result = getResult();` is just awful.
* `var` is just unable to declare the super-most type. Seriously, `Collection<t> collection = new ArrayList<>(); ...; collection.stream()...` is way bettter because I want to deal with the collection interface, not its implementation.
* `var` will probably have `val` as its counter-part (at least from what I remember the people want to have in a next release). Why breaking the eyes to blood to distinguish between vaR and vaL? Why not `let`? Moreover, values are just something totally different. `var` is not `final` by default.
* It cannot work with the diamond operator.

### Raw string literals

There is a [proposal](http://openjdk.java.net/jeps/326).
I always found funny and hurting to find some stuff like that in the code I'm going to work with.
Seriously, doesn't the current Java lexer allows breaking a string with new-lines?
If not, then why not just improve it within the current lexer similarly to C/C++ and not introduce a new literal just to serve a silly purpose?
Anyway, it's looks like PHP is invading the Java world.
Another thing here that always looked weak to me: should the indentation to become a string literal value?
Please, no.

### Switch expressions

There is another [proposal](http://openjdk.java.net/jeps/325).
Well, it doesn't look well-designed and would probably grow into another concept of pattern matching the most Java developers have never heard about.
`break 1` contradicts with `break one` where `one` is a label.
Introducing an arrow is probably another choice, but it also looks a sort of weird.
Why no `if` expressions or `while` expressions?
Why not a local nested _scope_ expressions so that it might be implemented in any manner, something like:

```java
final int i = @{
	switch ( type ) {
	case FOO: return 1;
	case BAR: return 2;
	case BAZ: return 3;
	default: throw new IllegalArgumentException(type);
};
```

I find the above much more extensible that this:

```java
final int i = switch ( type ) {
case FOO: break 1;
case BAR: break 2;
case BAZ: break 3;
default: throw new IllegalArgumentException(type);
}
```

or

```java
final int i = switch ( type ) {
case FOO -> 1;
case BAR -> 2;
case BAZ -> 3;
default -> -1;
}
```

Let-it-look-like-a-lambda design.
This is just an example but it can handle switches, iterations, branching, whatever you can put in a local nested scope.
But yeah, no people look a step ahead.
I believed that the ternary operator `?:` can be a nice thing to use everywhere, but I almost refused to use it and use it in very rare cases.
Even more, I think that such an above thing just requires to be extracted to another private method that can be reused, have a good name, and just take less room at the call site.

## What I really miss in Java

None of these improvements are solid.
All of them have serious design issues or contradictions.
And it's sad too see how Java turns from a verbose but gentle language into an ugly collect-these-all-features-no-matter-whether-they-are-fine monster.
Seriously, Java really needs not what can be handled by ANY Java IDE today, and Java devs seem to become more lazy now, but what is hard to implement:

* There is no `async`/`await`. `CompletableFuture`s are hard to implement and they cannot share the current function scope.
* There are no generators. I'm tired of writing complex iterators and spliterators every time I need something to be truly lazy.
* No string interpolation.
* No scope expressions.
* No LINQ or anything like that.
* There is no support for compile-time metaprogramming. I could just implement a for/else construct or a decorator pattern implementation, or whatever I could develop to simplify my code.

All of the above are not Java-instead-of-my-fav-IDE stuff, but fundamental improvements that could make Java less verbose using expressional power.
But Java seems to go a wrong way.
