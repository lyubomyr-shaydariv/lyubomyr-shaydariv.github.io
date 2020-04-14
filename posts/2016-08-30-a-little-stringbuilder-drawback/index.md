.. title: A little StringBuilder drawback
.. date: 2016-08-30 17:30:00 +0300
.. tags: java
.. category: java
.. type: text

Can you spot a mistake in the code below?

```java
StringBuilder sb1 = new StringBuilder("0");
StringBuilder sb2 = new StringBuilder('1');
System.out.println(sb1.append(sb2).toString());
```

<!-- TEASER_END -->

If you are attentive, you might spot it easily not even running the code.
I wasn't able to do it when I faced with such an issue a few weeks ago.
The result of the code above is really just a single zero and not `01`:

```
0
```

Why?
`"0"` and `'1'` are a string and a single character.
The first string builder `sb1` is constructed with an initial string `"0"`.
Whilst the second string builder `sb2` is constructed with some initial capacity.
Why?
Just because there are `StringBuilder` constructor overloads, and one of them takes declares a single `int` parameter.
`'1'` is a `char` so it will be converted to `int` silently initializing a `StringBuilder` instance with capacity of `49` characters.
So here are the overloads used in the code:

* [public StringBuilder(String str)](https://docs.oracle.com/javase/8/docs/api/java/lang/StringBuilder.html#StringBuilder-java.lang.String-)
* [public StringBuilder(int capacity)](https://docs.oracle.com/javase/8/docs/api/java/lang/StringBuilder.html#StringBuilder-int-)

My IntelliJ IDEA provides a nice set of inspections and intentions even for the Community Edition.
One of this inspections, _Replace String with Char_, warns about passing a string to methods that also may accept characters.
Fore example, the code below:

```java
return new StringBuilder()
	.append("(")
	.append(value)
	.append(")")
	.toString();
```

can be refactored to slightly different code:

```java
return new StringBuilder()
	.append('(')
	.append(value)
	.append(')')
	.toString();
```

I'm fine with such an intention since appending a single character is a bit faster than appending a string.
However, the following code (padded parenthesis):

```java
return new StringBuilder(" (")
	.append(value)
	.append(") ")
	.toString();
```

No additional `append` right after the constructor.
Once shorter delimiters are fine, it might be refactored manually:

```java
return new StringBuilder('(')
	.append(value)
	.append(')')
	.toString();
```

which does not work as it might be wanted to work.
Fortunately, IntelliJ IDEA reports a warning for the constructor (not for me; at least they say it has to).
Sometimes I think that it would be really great to have numerical types up-conversion explicit.
Even if slightly more code is necessary.
