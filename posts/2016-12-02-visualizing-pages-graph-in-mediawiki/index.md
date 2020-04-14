.. title: Visualizing pages graph in MediaWiki
.. date: 2016-12-02 11:53:00 +0300
.. tags: mediawiki, graphviz, imagemagick
.. category: mediawiki
.. type: text

> The original question was originally posted on April 11, 2016 at [MediaWiki Support Desk](https://www.mediawiki.org/wiki/Topic:T1w1xyhw7y8pb2g7)

I edit [MediaWiki](https://www.mediawiki.org/) based wikis.
MediaWiki is a very powerful system and besides the web-based administration toolkit there are also a big number of [maintenance scripts](https://www.mediawiki.org/wiki/Manual:Maintenance_scripts) that are accessible via shell.
One of the scripts is [`dumpLinks.php`](https://www.mediawiki.org/wiki/Manual:DumpLinks.php) that generates a plain text links dump.

<!-- TEASER_END -->

Plain text is very simple, but this simplicity may not be very convenient for certain purposes.
Let's convert it to a directed graph image to get a better visualization.

## `dumpLinks.php` output

First off, let's make a quick analysis of `dumpLinks.php` results.
The output is just a set of lines split into fields split by a space ([ASCII](https://en.wikipedia.org/wiki/ASCII) `0x20`; spaces in page names are replaced with `_`).
Let's group those fields into the _head_ and the _tail_, where _head_ is the first field containing a page name, and the _tail_ is the rest fields that are references to any other pages the _head_ refers to.
Here is an example:

```
Main_Page Foo Bar
Foo Bar
Bar Foo
```

So the output above is literally _The Main Page refers to both Foo and Bar, foo Refers to Bar, and Bar refers to Foo_.

## DOT

[DOT](https://en.wikipedia.org/wiki/DOT_(graph_description_language)) is a very simple graph description language and it can be easily generated and easily processed by other tools.
The plain text output above, in its DOT representation, may look as follows:

```
digraph wiki {
        "Main_Page" -> "Foo";
	"Main_Page" -> "Bar";
        "Foo" -> "Bar"
        "Bar" -> "Foo"
}
```

The `digraph` section describes a directed graph where all vertices are linked with arrows.
Really simple, right?

### `dumpLinks.php` output to DOT converter

The following script (`plain-to-dot.sh`) reads `dumpLinks.php` output from STDIN and writes the result into STDOUT:

```bash
#!/bin/bash

echo 'digraph wiki {'
while read ROW; do
	HEAD=$(echo "$ROW" | cut -d ' ' -f 1)
	TAIL=$(echo "$ROW" | cut -d ' ' -f 2-)
	# `cut` seems to return TAIL as HEAD if only one field is supplied
	if [ "$HEAD" != "$TAIL" ]; then
		for FIELD in $(echo -n "$TAIL" | tr " " "\n"); do
			echo -e "\t\"$HEAD\" -> \"$FIELD\";";
		done
	fi
done < "${1:-/dev/stdin}"
echo '}'
```

## The result

Once you have the graph, you can convert to [PostScript](https://en.wikipedia.org/wiki/PostScript) using [GraphViz](http://www.graphviz.org/), and then convert it to a PNG file using [ImageMagick](https://www.imagemagick.org/script/index.php):

```bash
php maintenance/dumpLinks.php \
	| ./plain-to-dot.sh \
	| dot -Tps \
	| convert ps:- png:- > wiki-links-graph.png
```

or alternatively use a pregenerated links file:

```bash
./plain-to-dot.sh < wiki-links.txt \
	| dot -Tps \
	| convert ps:- png:- > wiki-links-graph.png
```

And here is the result:

![Wiki links graph](/images/wiki-links-graph.png)


And here you can also download the [PostScript file](/downloads/wiki-links-graph.ps) the image above was generated from.
