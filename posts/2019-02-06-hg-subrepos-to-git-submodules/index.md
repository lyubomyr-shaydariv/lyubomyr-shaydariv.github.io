.. title: Mercurial subrepositories to Git submodules
.. date: 2019-02-06 23:16:00 +0300
.. tags: git, hg
.. category: scm
.. type: text

I didn't start with Git.
Mercurial has become my first DVCS.
Because of that, Mercurial actually ate my first source code.
I'm a huge Git fan now, and I switched all my single-repo repositories to Git using [this awesome tool](https://github.com/frej/fast-export).
Now, long few years later, the tool has [finally](https://github.com/frej/fast-export/issues/51) gotten the Mercurial subrepositories to Git submodules support, so now I'm able to migrate to Git completely.

<!-- TEASER_END -->

So what I did is the following steps:

### Convert all Mercurial subrepositories first

For each subrepository convert the subrepository like this:

```bash
git init SOME_GIT_SUBMODULE_NAME_N
cd SOME_GIT_SUBMODULE_NAME_N
hg-fast-export.sh -r SOME_HG_SUBREPO_PATH_N
git remote add origin SOME_REMOTE_URL_N
git push --mirror origin
cd ..
```

### Create the subrepositories to Git repositories mapping file

The mapping file is generally a key/value file where keys and values and enquoted with `"` and joined with `=`:

```
"dir1"="../SOME_GIT_SUBMODULE_NAME_1"
"dir2"="../SOME_GIT_SUBMODULE_NAME_2"
"dir3"="../SOME_GIT_SUBMODULE_NAME_3"
```

Note a few things here:

* The paths (the values) must be relative paths by the current design of the convertor.
* If the paths were moved during the lifetime of the super-repo, the previous subdirectories must also be specified as keys.
* If there's a repository that cannot be mapped anymore (say, a repository does not longer exist), consider use of an empty repository (not sure if it's a good idea though).

### Convert the super-repository using the mapping file

```bash
git init SOME_GIT_SUPER_REPO_NAME
cd SOME_GIT_SUPER_REPO_NAME
hg-fast-export.sh -r SOME_HG_SUPER_REPO_PATH --subrepo-map=MAPPING_FILE_PATH
cd ..
```

### Convert the relative submodule paths to remote repository URLs

`filter-branch` is a great tool that helped me a lot.
Now it can help one more time.
I resolved my own case using `sed`:

```bash
cd SOME_GIT_SUPER_REPO_NAME
git filter-branch -f --tree-filter "sed -i -e 's/url = SOME_LOCAL_PATH_TO_REPLACE/url = SOME_REPOSITORY_URL_TO_REPLACE_WITH/g' .gitmodules || true" -- --branches --tags
cd ..
```

which processes every commit in the repository moving branches and tags along.
If you have your submodules in subdirectories, then you might want to use a pipe-escaped form of `sed` not clashing with file path delimiters (i.e., `sed s|dir1/dir2|dir1/dir2|g`).
The `--tree-filter` may take some time to convert, but I didn't use the `--index-filter` option for simplicity.
