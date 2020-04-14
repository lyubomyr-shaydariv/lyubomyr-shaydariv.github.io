.. title: Efficient git filter-branch and --index-filter implementation
.. date: 2018-07-18 23:30:00 +0300
.. tags: git
.. category: scm
.. type: text

About two years ago I posted a post that described how you can use Git and Mercurial to create encrypted repositories.
Back to then, I claimed that the encryption cannot be changed and the encryption method will be constant for the entire repository lifecycle.
Well, sort of, from the user's perspective (at least if you don't use versioned `.gitattributes` that refer different crypto-filters).
Using and maintaining such repositories can be not a fun, and you might want to decrypt the whole repository some rainy day.

[`git filter-branch`](https://git-scm.com/docs/git-filter-branch) (a bit of a cryptic name, as well, yeah?) is a right tool.
It features a lot of filtering options that can transform the original repository from scratch, and we're going to use `--index-filter`...

<!-- TEASER_END -->

If you have ever faced `git filter-branch`, and your idea was file transormation, the first solution you might think of was probably [`--tree-filter`](https://git-scm.com/docs/git-filter-branch#git-filter-branch---tree-filterltcommandgt).
It's extremely simple.
But it's expensive, and even the official documentation says that.
The price for the simplicity is checking out every file to the working directory to process.
It's not cheap because every file has to be checked out and smudged before the `--tree-filter` comes into play.
Checking out may be slow, but the smudge filter can be even slower.
Even the files that were not changed in latest commits must be checked out.

Okay, the Git has another option, called [`--index-filter`](https://git-scm.com/docs/git-filter-branch#git-filter-branch---index-filterltcommandgt) that is designed to operate the index.
Knowing that the index is something that does not necessarily need checking out, the first attempt for using the index filter could be naive as well: the files should be checked out and transformed.
If you miss to modify a file in the index, it will remain unchanged for that commit, because Git commits are _snapshots_ that can be thought of a list of references to blobs.
Say, if you decide to decrypt an encrypted repository, at some steps, if not all files are processed in the filter, the history will contain undecrypted files that's probably not the idea.
(By the way, the `--index-filter` example in the Git documentation assumes that you know what the index is and simply demonstrates `git rm` scenario to delete files from the repository).

Both `--tree-filter` and `--index-filter` (if the latter is used for file transformation purposes) are awful time killers.
For example, decrypting a repository using the `--tree-filter` must decrypt every blob in the repository even if blobs were decrypted already (remember the snapshots).
Now imagine to have an encrypted 1000-commit repository where you have ~1000 files but only some of them are modified throughout the history.
The clean/smudge filter will decrypt the file over and over wasting time extremely (say hello to _O(n^2)_).

What's then?
`--index-filter` is the right way to go.
How do we deal with the snapshots then?
Not an issue: just reuse processed blobs and fix blob references.
This will save you a lot of time.

I've implemented a simple `git-rewrite-all-files.sh` script and I'll just put it as it is with all the comments it has.

```bash
#!/bin/bash

set -e

# Verbosity level
VERBOSE=
# A command to process each blob
BLOB_FILTER=

while [ -n "$1" ]; do
	case "$1" in
	-v)
		VERBOSE=1
		;;
	*)
		BLOB_FILTER="$1"
		;;
	esac
	shift
done

if [[ -z "$BLOB_FILTER" ]]; then
	echo 'no blob filter specified'
	exit 1
fi

# The script context directory is repo/.git-rewrite/t, but let's change the directory to a predifed directory that can handle the purpose as well
cd ../map

[ -n "$VERBOSE" ] && echo

# Process each file in the current commit
git ls-files --stage | while read LINE; do
	# Parse mode, object and file
	MODE="${LINE:0:6}"
	OBJECT="${LINE:7:40}"
	FILE="${LINE:50}"
	# The object file contains a mapping between the old and the new object, where:
	# - the filename is the old object (key)
	# - the file content is the new object (value)
	if [[ ! -f "$OBJECT" ]]; then
		[ -n "$VERBOSE" ] && echo -en "\tFILTER $OBJECT $FILE "
		# Extract the old object, filter it out, and write it to the objects database returning its new object hash
		git cat-file blob "$OBJECT" | GIT_FILE="$FILE" eval "$BLOB_FILTER" | git hash-object -w --stdin > "$OBJECT"
		[ -n "$VERBOSE" ] && echo "$(cat $OBJECT)"
	fi
	# Map the objects
	NEW_OBJECT="$(cat $OBJECT)"
	[ -n "$VERBOSE" ] && echo -e "\tUSE $OBJECT $NEW_OBJECT"
	# Replace the file blob by its new object
	git update-index --add --cacheinfo "$MODE,$NEW_OBJECT,$FILE"
done
```

As you can see, the filter script lazily converts every blob, and once it detects the blob was already converted, it simply tells Git to reuse a blob that now has another object hash.
That's all.

Examples of use:

* Append `EOF: <filename>` to each file in each commit:

```bash
git filter-branch -f --index-filter '<PATH_TO_THE_SCRIPT>/git-rewrite-all-files.sh "cat; echo EOF: \$GIT_FILE;"' -- --branches --tags
```

* Decrypt entire repository using `openssl` but let the latter fail gracefully:

```bash
git filter-branch -f --index-filter '<PATH_TO_THE_SCRIPT>/git-rewrite-all-files.sh "openssl enc -d -aes256 -k <PASSWORD> || true;"' -- --branches --tags
```

Well, yes, decrypting a repository is just a particular use case and your scripts can be way more advanced.
