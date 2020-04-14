.. title: Unique code search algorithm rewritten in almost pure ReQL
.. date: 2017-11-22 19:10:00 +0300
.. tags: algorithms, javascript, rethinkdb, reql
.. category: rethinkdb
.. type: text

In my previous post regarding generating the unique codes out of the free codes.
That implementation was based on a JavaScript function that accepted a strategy to be invoked each time a binary search iteration is in progress.
I also mentioned that due this it will do _log n_ queries to the RethinkDB database, and I assumpted that it can be fully ReQL based.
Yes, it can be executed fully on the RethinkDB side being re-implemented in pure ReQL.
Almost.
Let's see how.

<!-- TEASER_END -->

Thanksfully, RethinkDB supports two basic functions like [`range`](https://www.rethinkdb.com/api/javascript/range/) and [`fold`](https://www.rethinkdb.com/api/javascript/fold/).
How could they help it?
Well, it's clear about `range`: it just serves the purpose of making iterations.
What about `fold`?
Using the `fold` iteration it can share the state between the iterations generated from the `range` above.

Let's see.

```javascript
const findFreeValue = (min, max, hasFree) =>
	r.branch(
		// if min/max difference is at least 1 (min is closed, max is open) ...
		r(max).sub(min).eq(1),
		// ... then report no values
		r.error('all values in use'),
		// ... else do search with maximum number of steps required for the binary search
		r.range(Math.ceil(Math.log2((max - min))))
			.fold({l: min, r: max}, (a, _) =>
				a.do(() => {
					// get the floored middle between l and r
					const m = a('r').sub(a('l')).div(2).floor().add(a('l'));
					return r.branch(
						// randomly get the "first" strategy
						r.random().lt(0.5),
						// use left-first then right-second
						r.branch(
							// if there's anything free between l and m, ...
							hasFree(a('l'), m),
							// ... then move the right to the middle
							r.do(() => a.merge({r: m})),
							// else if there's anything free between l and r, ...
							hasFree(m, a('r')),
							// ... then move the left to middle
							r.do(() => a.merge({l: m})),
							// ... otherwise nothing left
							r.error('all values in use')
						),
						// otherwise use right-first then left-second
						r.branch(
							hasFree(m, a('r')),
							r.do(() => a.merge({l: m})),
							hasFree(a('l'), m),
							r.do(() => a.merge({r: m})),
							r.error('all values in use')
						)
					);
				})
			)
			// l always points to a free value
			.do(a => a('l'))
	);
```

Note that we must calculate the number of iterations before the execute the query, because we cannot break the loop implemented with `fold`.
It would be awesome if `fold` could do this.
Also, this why I said "almost" above regarding the algorithm fullness in ReQL: RethinDB does not support logarithms, so we're cheating there a bit.
Next, all the `fold` operation is responsible for is sharing the state of two variables, `l` and `r`, between iterations.
Note that we can update the accumulator state on each iteration by applying the `merge` operator.

Now, this is how it can be used after all:

```javascript
findFreeValue(256, (codeL, codeR) => r.db('test')
	.table('codes')
	.between(codeL, codeR, { index: 'code' })
	.count()
	.do(count => r(codeR).sub(codeL).gt(count))
);
```

I think it's pretty cool and saves the amount of queries suggested in the previous approach.
Sadly, this is the second time I need to break the `fold` and have no ReQL operator to do it.
