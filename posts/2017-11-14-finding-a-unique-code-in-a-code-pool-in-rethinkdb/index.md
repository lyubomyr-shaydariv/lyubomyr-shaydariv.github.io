.. title: Finding a unique code in a code pool in RethinkDB
.. date: 2017-11-14 23:37:00 +0300
.. tags: algorithms, javascript, rethinkdb, reql
.. category: rethinkdb
.. type: text

My recent project had a lot of fun to work with.
And, as I said before, working with RethinkDB was fun too.
Since that project implemented a virtual file system, it also had to share a file by a unique file by a unique code (say, `kl2890aj` or `2opOZxk1`).

How did it share a file by a unique code similarly to what Dropbox or YouTube do?

<!-- TEASER_END -->

The algorithm it used is simply a [binary search algorithm](https://en.wikipedia.org/wiki/Binary_search_algorithm) with average complexity of _O(log n)_.
That's pretty cool, because we can allocate a huge number of available codes, and it would only take a few steps to find the first free random unique code.
I will not describe how binary search algorithm works, but I'll implement it in JavaScript using `async`/`await` because it would be aligned with asynchronous code that is required if working with RethinkDB.

### Codes and raw codes

First of all, let's describe an alphabet for the codes.
The alphabet must consist of all possible values characters available in the codes.
Also we need to distinct between codes and _raw_ codes.
Raw codes are just codes (integer values unlike strings for "regular" codes) in decimal base whilst regular codes can consist of digits, letters and other characters like Base64.
Why do we need distinct between codes and raw codes?
Because we must be able to compare raw codes.
But RethinkDB allows string comparison for strings like `eq`, `ne`, `lt`, `gt`, `le`, and `ge`.
Well the alphabet may consist of characters that are not necessarily ordered by their character set positions, therefore we can define our custom ordering.
Note that having raw codes would require storing raw codes in the database as well, so it's your choice of the way you want to represent unique codes.

The first function is simple and converts a given number according to the given alphabet.
For example, `rawCodeToCode(255, '0123456789ABCDEF')` returns `"FF"`.

```javascript
const rawCodeToCode = (rawCode, alphabet, padLength = undefined) => {
	if ( typeof rawCode !== 'number' ) {
		throw new Error(`Raw code ${rawCode} is not a number (actual type = ${typeof rawCode})`);
	}
	let result = '';
	let reducedCode = rawCode;
	const base = alphabet.length;
	while ( reducedCode > 0 ) {
		const index = reducedCode % base;
		result = alphabet.charAt(index) + result;
		reducedCode = Math.floor(reducedCode / base);
	}
	if ( padLength ) {
		return lodash.padStart(result, padLength, alphabet.charAt(0));
	}
	return result;
};
```

`codeToRawCode`, unlike the previous function, does a reverse operation: it computes a number that corresponds to the given alphabet by a code.
For example, `"FF"` must be converted back to `255` in this case.

```javascript
const codeToRawCode = (code, alphabet) => {
	if ( typeof code !== 'string' ) {
		throw new Error(`Code ${code} is not a string (actual type = ${typeof code})`);
	}
	let rawCode = 0;
	const base = alphabet.length;
	for ( let k = 1, i = code.length - 1; i >= 0; k *= base, i-- ) {
		const index = alphabet.indexOf(code.charAt(i));
		if ( index === -1 ) {
			throw new Error(`Code ${code} has characters that are not declared in the alphabet ${alphabet}`);
		}
		rawCode += index * k;
	}
	return rawCode;
};
```

### Binary search algorithm implementation

The implementation is trivial.
The only significant thing here is that it's implemented using `async`/`await` therefore `hasFreeAsync` can return a boolean `Promise` that can be fetched from RethinkDB.
Why does it use `max` rather than begin-inclusive and end-exclusive?
Just consider the begin-inclusive is always `0` whilst end-exclusive equals `max`.
It just simplifies the things.
For example, `max` should be `256` from the example above, since `255` is the last available code in the code pool.

```javascript
const findUniqueRawCodeAsync = async(max, hasFreeAsync) => {
	if ( max === 0 ) {
		throw new Error('all values in use');
	}
	let left = 0;
	let right = max;
	for ( ; ; ) {
		if ( right - left === 1 ) {
			return left;
		}
		const middle = left + Math.floor((right - left) / 2);
		if ( Math.random() < 0.5 ) {
			if ( await hasFreeAsync(left, middle) ) {
				right = middle;
			} else if ( await hasFreeAsync(middle, right) ) {
				left = middle;
			} else {
				throw new Error('all values in use');
			}
		} else {
			if ( await hasFreeAsync(middle, right) ) {
				left = middle;
			} else if ( await hasFreeAsync(left, middle) ) {
				right = middle;
			} else {
				throw new Error('all values in use');
			}
		}
	}
};
```

### RethinkDB

Since the binary search algorithm is complete, it can be combined with a RethinkDB database.
I'm not really sure if the algorithm can be implemented in ReQL (it looks like it can), so the algorithm would use a strategy via `hasFreeAsync`.
Also note that a table that contains the codes should have an index for the codes column in order to make the search efficient (otherwise it can run extremely long).

```javascript
const hasFreeQuickCodesAsync = (leftRawCode, rightRawCode) =>
	r.db(/*your database here*/)
		.table(/*your table here*/)
		.between(leftRawCode, rightRawCode, { index: 'raw_code' })
		.count()
		.run(connection)
		.then(count => rightRawCode - leftRawCode > count);
// ...
findUniqueRawCodeAsync(256, hasFreeQuickCodesAsync)
	.then((rawCode) => {
		const code = rawCodeToCode(rawCode, '0123456789ABCDEF');
		console.log(`Free code: ${code}`);
		return code;
	})
	.then((code) => {
		// save the code back to the database
		// the next time findUniqueRawCodeAsync(256, hasFreeQuickCodesAsync) is invoked, it will be occupied
	})
;
```
