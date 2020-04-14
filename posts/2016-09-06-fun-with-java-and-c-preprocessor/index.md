.. title: Making fun with Java and C preprocessor
.. date: 2016-09-06 10:30:00 +0300
.. tags: java
.. category: java
.. type: text

Sometimes, usually on Fridays, there is nothing to do.
And I always missed a very cool feature of meta-programming in Java.
Well, Java is not very excited about making changes in the core language, so we can make some fun with meta-programming ourselves.
Let's take a look at a Java and [C preprocessor](https://gcc.gnu.org/onlinedocs/cpp/) example.
And remember, it's just for fun and nothing else. :)

<!-- TEASER_END -->

Java features a cool thing called [annotation processing tools](http://docs.oracle.com/javase/7/docs/technotes/guides/apt/).
Some of compiler plugins like [Lombok](https://projectlombok.org/) and [lombok-pg](https://github.com/peichhorn/lombok-pg) are built on top of the feature.
They are more complex than merely a C preprocessor, so we're about to make some old-school here.

One of my favorite never happened to be implemented features in Java is `foreach/otherwise`.
I even asked [this feature](https://github.com/dotnet/roslyn/issues/134) to be added to C#, however it was rejected by the Roslyn team.
More meta-programming friendly languages like [Nemerle](http://nemerle.org) implement [this macro](https://github.com/rsdn/nemerle/blob/master/ncc/testsuite/positive/foreach-otherwise-macro.n).
Is it possible to implement such a feature in Java?
Well, yes.
We will not use [FreeMarker](http://freemarker.org/) or any other template processing engine since the C preprocessor is the goal.

First, let's define the `foreach/otherwise` macro.

```c
#define __LF__ /*
	*/

#define FOR_EACH_ARRAY_OR_OTHERWISE(A, i) \
	if ( A.length > 0 ) { __LF__\
		for ( i = 0; i < A.length; i++ ) {
#define FOR_EACH_OTHERWISE } } else {
#define FOR_EACH_END  }
```

As the C preprocessor macro definitions are really straight-forward, they are obvious:

* `__LF__` &mdash; since the C preprocessor does not support new lines, it's a [trick](http://stackoverflow.com/questions/2271078/how-to-make-g-preprocessor-output-a-newline-in-a-macro) that generates comments across two lines;
* `FOR_EACH_ARRAY_OR_OTHERWISE` &mdash; this is the most "magic" thing here: just accept two parameters (an array and the name of an iterator variable) and generate some source code checking for array length and making decision on which branch to go with;
* `FOR_EACH_OTHERWISE` &mdash; if the given array is empty, then just do nothing;
* `FOR_EACH_END` &mdash; and finalize the macro construction.

Let's try them all:

```java
#include "Base.javainc"

import static java.lang.System.out;

public final class HelloWorld {

	private HelloWorld() {
	}

	public static void main(final String... args) {
		int i;
		FOR_EACH_ARRAY_OR_OTHERWISE(args, i)
			out.print("Hello ");
			out.println(args[i]);
		FOR_EACH_OTHERWISE
			out.println(":(");
		FOR_EACH_END
	}

}
```

Well, not that Java-like, but we're metaprogramming, and that's not an issue. :)
Then let's run the C preprocessor for the C preprocessor-applied almost-Java source:

```
cpp -traditional-cpp -P -CC HelloWorld.javacpp > HelloWorld.java
```

The command line options, generally and roughly speaking, are:

* `-traditional-cpp` &mdash; try to keep the original formatting;
* `-P` &mdash; do not generate line markers (such a line marker starts with `#` and we don't need it in Java code);
* `-CC` &mdash; keep comments thus letting to apply the new line trick.

The command above generates the following `HelloWorld.java` content:

```java
/*
... C preprocessor comments go here ...
 */

public final class HelloWorld {

	private HelloWorld() {
	}

	public static void main(final String... args) {
		int i;
		if ( args.length > 0 ) { /*
        */		for (  i = 0;  i < args.length;  i++ ) {
			out.print("Hello ");
			out.println(args[i]);
		} } else {
			out.println(":(");
		}
	}

}
```

The indentations are broken, but it's still fully legal Java code.
If you don't bother pretty print, you might want to collapse the macro definition to one line, and remove the `__NL__` macro use along with the `-CC` key discarding the preprocessor generated comments.

```c
#define FOR_EACH_ARRAY_OR_OTHERWISE(A, i) if ( A.length > 0 ) { for ( i = 0; i < A.length; i++ ) {
```

Let's just compile it and run:

```
javac HelloWorld.java
java HelloWorld
java HelloWorld YOU ME
```

The latter two `java` commands will produce `:(` and sequential `Hello YOU` and `Hello ME` respectively.
So the `foreach/otherwise` macro is done.

Note that this idea is not that crazy.
The [fastutils](https://github.com/vigna/fastutil) library uses the C preprocessor to generate all specialiazations for Object and all primitive types, and [the defined macros](https://github.com/vigna/fastutil/tree/master/drv) help it a lot.
