.. title: Making a unique filename by adding a copy number in RethinkDB
.. date: 2017-11-14 14:13:00 +0300
.. tags: rethinkdb, reql
.. category: rethinkdb
.. type: text

[RethinkDB](https://www.rethinkdb.com/) is a beautiful document-oriented database I was lucky to use for my most recent project.
Unfortunately, I don't work for that project anymore, but I would like to share some ideas I implemented while working on that project.

That project implemented a virtual file system table in a RethinDB database, and one of the requirements while creating a new file entry was making its filename unique.
A usual filename generated chain named as if you downloading a same-named file: `file.ext`, `file (1).ext`, `file (2).ext` and so on.

<!-- TEASER_END -->

First of all, let's drop a case when we download all filenames and calculate the first free copy number at the backend like Java or JavaScript/nodejs.
Also we could implement such a search using a mixed scenario: the search algorithm in Java or JavaScript where the search algorithm strategy could be implemented as a single database query execution.

Fortunately, ReQL is powerful enough to let such an algorithm to be implemented completely using ReQL facilities.
I'll implement the query in JavaScript since it can be easily tested in Data Explorer.

Second, let me describe a simple algorithm with _O(n)_ complexity:

1. Get all filenames that have the following naming pattern: `file.ext` or `file (n).ext` where _n >= 1_.
2. Extract the copy number _n_ (if no copy number is present, assume it's _0_).
3. Create a sorted list _a_ of unique elements from all _n_ values.
4. If we don't have the 0 value in that list, we don't need to add the copy number to the filename, therefore the algorithm can terminate.
5. Iterate from the first element of the list _a[n]_, and check every next value _a[n + 1]_: if their difference is 1, then no free slot is available, otherwise there should be a free value slot.

For example:

1. We have a set of `file (4).ext`, `file (2).ext`, `file.ext`, and `file (1).ext`.
2. It gets transformed in _[4, 2, 0, 1]_.
3. When it gets sorted, it becomes _[0, 1, 2, 4]_.
4. The list has _0_, so we must search further.
5. Having that, skip the first value, we calculate the difference of _1_ and _0_ which is _1_ (meaning that _1_ is occupied), the difference of _2_ and _1_ which is _1_ (meaning that _2_ is occupied), the difference of _4_ and _2_ which is _2_ (meaning that _2_ is free).

Here is a ReQL implementation:

```javascript
r.db(/*your database name here*/)
	.table(/*your table name here*/)
	.filter(/*your filter search criteria here*/)
	// match each filename against the given pattern
	.map(file => file('fileName').match(`^${quoteRx(filename)}(?: \\((\\d+)\\))?${quoteRx(extension)}$`))
	// extract the copy number group
	.getField('groups')
	.map(group => group.nth(0))
	// if the filename matches the pattern, but no copy number found, assume it's just 0 since we start copies from 1
	.map(value => r.branch(value.ne(null), value('str').coerceTo('number'), 0))
	// make them all unique
	.distinct()
	// and ensure they are sorted from min to max (not sure if r.distinct() can do it itself)
	.orderBy(n => n)
	// analyze the obtained copy numbers
	.do(array => r.branch(
		// if the found array is empty or there are any copy number greater than 1 (that we start from)
		array.count().eq(0).or(array.nth(0).ge(1)),
		// then return -1 as "not found"
		-1,
		// else, knowing that we have at least 1-element array whose first element is not 0
		array.skip(1)
		// start the folding setting its initial minimum value to the first array element value (it's the minimum already)
		// and check if every such a pair difference is 1:
		// * if the difference is 1, then we've found a greater value and must proceed to the next iteration
		// * otherwise, return the minimum (TODO is it possible to break fold iterations in ReQL?)
		.fold(array.nth(0), (minimum, current) => r.branch(current.sub(minimum).eq(1), current, minimum))
		// and increase the last found by one therefore giving the first "free" slot
		.add(1),
	))
```

One more note about `quoteRx()`.
This is a function that quotes any string in order to disable all regular expression characters represented by that string.
We cannot just prepend each character by a backslash `\` because it will cause a runtime error when RethinkDB will parse the regular expression.
RethinkDB [`match()`](https://www.rethinkdb.com/api/javascript/match/) operator uses [RE2 syntax](https://github.com/google/re2/wiki/Syntax) and I couldn't find any quote function implementation there.
I hope the following function covers RE2 syntax nicely (it only prepends the characters that have special meaning as a regular expression symbol):

```javascript
const metacharacters = new Set([
  '\\', '+', '*', '?', '[', '^', ']',
  '$', '(', ')', '{', '}', '=', '!',
  '<', '>', '|', ':', '-',
]);

const quoteRx = rx => rx.replace(/(.)/g, ch => (metacharacters.has(ch) ? `\\${ch}` : ch));
```
