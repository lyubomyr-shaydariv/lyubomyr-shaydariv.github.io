.. title: Highlight with grc
.. date: 2016-12-03 13:30:00 +0300
.. tags: grc, ant, diff, maven, mysql, subversion
.. category: misc
.. type: text

I use my terminal emulators a lot.
I just love [command-line interface](https://en.wikipedia.org/wiki/Command-line_interface) because I really feel much more freedom than I have in GUI applications.
However the most command-line applications do not highlight their output making it hard to read.
Let's fix it?

<!-- TEASER_END -->

A few years ago I discovered a nice tool, [Generic Colouriser](http://korpus.juls.savba.sk/~garabik/software/grc.html) (mirror at [GitHub](https://github.com/pengwynn/grc)), that reads from standard input and generates colorized output according to the configured rules.
The general syntax of `grc` is as follows:

```
grc -c <configuration_file> -s -e <colorized_output_command>
```

where the options are:

* `-c` - configuration file
* `-s` - colorize stdout
* `-e` - colorize stderr

Since that time I made a small list of tools I work with the most.
So here it is.

## ant

Bash shortcut:

```bash
function ANT {
	grc -c ~/ant.grc -s -e ant "$@"
}
```

`ant.grc`:

```
regexp=^(\S.*):
colours=green
-

regexp=^\s*\[[a-z0-9\-]+\]
colours=yellow
-

regexp=^BUILD SUCCESSFUL
colours=bold white on_green
-

regexp=^BUILD FAILED
colours=bold white on_red
```

Sample output:

<pre style="background-color: black; color: white;">
<span style="color: green;">Buildfile:</span> /home/lsh/tmp/build.xml

<span style="color: green;">all:</span>
     <span style="color: yellow;">[echo]</span> grc test

<span style="background-color: green; font-weight: bolder;">BUILD SUCCESSFUL</span>
<span style="color: green;">Total time:</span> 3 seconds
</pre>

## diff (unified)

Bash shortcut:

```bash
function DIFF {
	grc -c ~/diff.grc -s -e diff "$@"
}
```

`diff.grc`:

```
regexp=^@@\s.*\s@@$
colours=bold black
-

regexp=^Index:\s.*$
colours=white on_blue
-

regexp=^=+$
colours=white on_blue
-

regexp=^-.*$
colours=red
-

regexp=^\+.*$
colours=green
-

regexp=^---\s.*$
colours=bold red
-

regexp=^\+\+\+\s.*$
colours=bold green
```

Sample output:

<pre style="background-color: black; color: white;">
<span style="color: red; font-weight: bolder;">--- a   2016-12-03 12:20:25.928610790 +0200</span>
<span style="color: green; font-weight: bolder;">+++ b   2016-12-03 12:20:43.628610011 +0200</span>
<span style="color: darkgray; font-weight: bolder;">@@ -1,4 +1,7 @@</span>
<span style="color: green;">+&lt;p&gt;</span>
 Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
 Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.
 Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur.
 Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
<span style="color: green;">+&lt;/p&gt;</span>
<span style="color: green;">+</span>
</pre>

## Maven

Bash shortcut:

```bash
function MVN {
	grc -c ~/mvn.grc -s -e mvn "$@"
}
```

`mvn.grc`:

```
regexp=^\[\s*FATAL\s*\].*
colours=bold white on_red
-

regexp=^\[\s*ERROR\s*\].*
colours=red
-

regexp=^\[\s*WARN(ING)?\s*\].*
colours=yellow
-

regexp=^\[\s*INFO\s*\].*
colours=green
-

regexp=^\[\s*DEBUG\s*\].*
colours=blue
-

regexp=^\[\s*TRACE\s*\].*
colours=bold black
-

regexp=BUILD FAILURE
colours=white on_red
-

regexp=BUILD SUCCESS
colours=bold white on_green
-

regexp=^(\tat\s|Caused by: |\t\.\.\. \d+ more).*
colours=red
```

Sample output:

<pre style="background-color: black; color: white;">
<span style="color: green;">[INFO] Scanning for projects...</span>
<span style="color: green;">[INFO]</span>
<span style="color: green;">[INFO] ------------------------------------------------------------------------</span>
<span style="color: green;">[INFO] Building grc test 0</span>
<span style="color: green;">[INFO] ------------------------------------------------------------------------</span>
<span style="color: green;">[INFO]</span>
<span style="color: green;">[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ grc test ---</span>
<span style="color: yellow;">[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!</span>
<span style="color: green;">[INFO] skip non existing resourceDirectory /home/lsh/tmp/src/main/resources</span>
<span style="color: green;">[INFO]</span>
<span style="color: green;">[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ grc test ---</span>
<span style="color: green;">[INFO] No sources to compile</span>
<span style="color: green;">[INFO]</span>
<span style="color: green;">[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ grc test ---</span>
<span style="color: yellow;">[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!</span>
<span style="color: green;">[INFO] skip non existing resourceDirectory /home/lsh/tmp/src/test/resources</span>
<span style="color: green;">[INFO]</span>
<span style="color: green;">[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ grc test ---</span>
<span style="color: green;">[INFO] No sources to compile</span>
<span style="color: green;">[INFO]</span>
<span style="color: green;">[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ grc test ---</span>
<span style="color: green;">[INFO] No tests to run.</span>
<span style="color: green;">[INFO]</span>
<span style="color: green;">[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ grc test ---</span>
<span style="color: yellow;">[WARNING] JAR will be empty - no content was marked for inclusion!</span>
<span style="color: green;">[INFO] Building jar: /home/lsh/tmp/target/grc test-0.jar</span>
<span style="color: green;">[INFO] ------------------------------------------------------------------------</span>
<span style="color: green;">[INFO] <span style="background-color: green; color: white; font-weight: bolder;">BUILD SUCCESS</span></span>
<span style="color: green;">[INFO] ------------------------------------------------------------------------</span>
<span style="color: green;">[INFO] Total time: 3.626 s</span>
<span style="color: green;">[INFO] Finished at: 2016-12-03T14:33:58+02:00</span>
<span style="color: green;">[INFO] Final Memory: 11M/89M</span>
<span style="color: green;">[INFO] ------------------------------------------------------------------------</span>
</pre>

## MySQL

Bash shortcut:

```bash
function MYSQL {
	mysql --pager='grcat ~/mysql.grc | less -RSFX' "$@"
}
```

`mysql.grc`:

```
# default word color
regexp=.+
colours=white
-

# table borders
regexp=[+\-]+[+\-]|[|]
colours=green
-

# NULLs
regexp=\bNULL\b
colours=on_blue
-

# data in ( ) and ' '
regexp=\([\w\d,']+\)
colours=blue
-

# numeric
regexp=\s[\d\.]+(\s|.$)
colours=yellow
-

# date or time
regexp=(\d{4}-\d{2}-\d{2}|\d{2}:\d{2}:\d{2})
colours=cyan
-

# hashes
regexp=\b\*?[0-9A-F]{6,}\b
colours=black on_yellow
-

# IP v4 address
regexp=(\d{1,3}\.){3}\d{1,3}(:\d{1,5})?
colours=cyan
-

# schema
regexp=`\w+`
colours=yellow
-

# email
regexp=[\w\.\-_]+@[\w\.\-_]+
colours=magenta
-

# row delimeter when using \G key
regexp=[*]+.+[*]+
count=stop
colours=white
-

# column names when using \G key
regexp=^\s*\w+:
colours=white
```

Sample output:

<pre style="background-color: black; color: white;">
<span style="color: green;">+-----------------------+---------------+------+-----+---------+-------+</span>
<span style="color: green;">|</span> Field                 <span style="color: green;">|</span> Type          <span style="color: green;">|</span> Null <span style="color: green;">|</span> Key <span style="color: green;">|</span> Default <span style="color: green;">|</span> Extra <span style="color: green;">|</span>
<span style="color: green;">+-----------------------+---------------+------+-----+---------+-------+</span>
<span style="color: green;">|</span> Host                  <span style="color: green;">|</span> char<span style="color: blue;">(60)</span>      <span style="color: green;">|</span> NO   <span style="color: green;">|</span> PRI <span style="color: green;">|</span>         <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Db                    <span style="color: green;">|</span> char<span style="color: blue;">(64)</span>      <span style="color: green;">|</span> NO   <span style="color: green;">|</span> PRI <span style="color: green;">|</span>         <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> User                  <span style="color: green;">|</span> char<span style="color: blue;">(16)</span>      <span style="color: green;">|</span> NO   <span style="color: green;">|</span> PRI <span style="color: green;">|</span>         <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Select_priv           <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Insert_priv           <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Update_priv           <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Delete_priv           <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Create_priv           <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Drop_priv             <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Grant_priv            <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> References_priv       <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Index_priv            <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Alter_priv            <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Create_tmp_table_priv <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Lock_tables_priv      <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Create_view_priv      <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Show_view_priv        <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Create_routine_priv   <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Alter_routine_priv    <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Execute_priv          <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Event_priv            <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">|</span> Trigger_priv          <span style="color: green;">|</span> enum<span style="color: blue;">('N','Y')</span> <span style="color: green;">|</span> NO   <span style="color: green;">|</span>     <span style="color: green;">|</span> N       <span style="color: green;">|</span>       <span style="color: green;">|</span>
<span style="color: green;">+-----------------------+---------------+------+-----+---------+-------+</span>
</pre>

## Subversion

Bash shortcut (this shortcut also adds pagination and specifies different rules for different `svn` commands):

```bash
function SVN {
	local RULES=~/svn.grc
	local PAGED=
	case "$1" in
		"blame")
			PAGED=yes
			;;
		"diff")
			RULES=~/diff.grc
			PAGED=yes
			;;
		"log")
			PAGED=yes
			;;
		"st")
			PAGED=yes
			;;
		"status")
			PAGED=yes
			;;
		*)
			;;
	esac
	if [ "$PAGED" == "yes" ]; then
		grc -c $RULES -s -e svn "$@" | nl | less -rFX
	else
		grc -c $RULES -s -e svn "$@"
	fi
}
```

`svn.grc`:

```
##################
## FIRST COLUMN ##
##################

# ? - untracked
regexp=^\?.*
colours=bold black
-

# ! - missing
regexp=^\!.*
colours=white on_red
-

# A - added
regexp=^(A)... (.*)
colours=,bold cyan,cyan
-

# C - conflicted
regexp=^C... .*
colours=black on_yellow
-

# D - deleted
regexp=^(D)... (.*)
colours=,bold red,red
-

# G - merged (during svn update)
regexp=^(G)... (.*)
colours=,bold yellow,yellow
-

# I - ignored
regexp=^I... .*
colours=bold black
-

# M - modified
regexp=^(M)... (.*)
colours=,bold green,green
-

# R - replaced
regexp=^(R)... (.*)
colours=,bold magenta,magenta
-

# U - updated (during svn update)
regexp=^(U)... (.*)
colours=,bold green,green
-

# X - external
regexp=^X... .*
colours=magenta
-

###################
## SECOND COLUMN ##
###################

# M - modified properties
regexp=^.(M).. .
colours=,bold on_green
-

# C - conflicted properties
regexp=^.(C).. .
colours=,bold on_yellow
-

##################
## THIRD COLUMN ##
##################

# L - locked
regexp=^..(L). .
colours=bold red
-

###################
## FOURTH COLUMN ##
###################

# + - copied
regexp=^...(\+)
colours=,bold cyan
-

##################
## FIFTH COLUMN ##
##################

# S - switched
regexp=^....(S)
colours=,green
-

##################
## SIXTH COLUMN ##
##################

# K - locked
regexp=^.....(K)
colours=,red
-

# O - locked in another working copy
regexp=^.....(O)
colours=,bold red
-

# T - stolen lock
regexp=^.....(T)
colours=,bold green
-

# B - broken lock
regexp=^.....(B)
colours=,bold magenta
-

##############
## SVN INFO ##
##############

regexp=^(Path|Working Copy Root Path|URL|Relative URL|Repository Root|Repository UUID|Revision|Node Kind|Schedule|Last Changed Author|Last Changed Rev|Last Changed Date):(.*)
colours=,cyan,bold cyan
-

#############
## SVN LOG ##
#############

regexp=^-{6,}
colours=green
-

regexp=^(r\d+) \|
colours=,bold yellow
-

###############
## SVN BLAME ##
###############

regexp=^(\d+)\s+([0-9a-zA-Z\-\.]+)\s+
colours=,yellow, green
```

Sample output:

<pre style="background-color: black; color: white;">
     1  <span style="background-color: blue; color: white; font-weight: bolder;">Index: pom.xml</span>
     2  <span style="background-color: blue; color: white; font-weight: bolder;">===================================================================</span>
     3  <span style="color: red; font-weight: bolder;">--- pom.xml   (revision 1772463)</span>
     4  <span style="color: green; font-weight: bolder;">+++ pom.xml   (working copy)</span>
     5  @@ -24,7 +24,7 @@
     6     &lt;parent&gt;
     7       &lt;groupId&gt;org.apache.maven.enforcer&lt;/groupId&gt;
     8       &lt;artifactId&gt;enforcer&lt;/artifactId&gt;
     9  <span style="color: red;">-    &lt;version&gt;1.4.1&lt;/version&gt;</span>
    10  <span style="color: green;">+    &lt;version&gt;1.4.2&lt;/version&gt;</span>
    11     &lt;/parent>
    12
    13     <groupId>org.apache.maven.plugins</groupId>
</pre>

<pre style="background-color: black; color: white;">
     1  <span style="color: cyan;"><span style="font-weight: bolder;">A</span>       README</span>
     2  <span style="color: green;"><span style="font-weight: bolder;">M</span>       pom.xml</span>
</pre>
