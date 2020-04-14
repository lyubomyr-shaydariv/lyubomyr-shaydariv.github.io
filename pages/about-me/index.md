.. title: About me
.. slug: about-me
.. date: 2018-10-07 15:03:40 UTC+02:00
.. tags: about
.. category:
.. link:
.. description:
.. type: text

I design and I code.
Passionate about programming and software development in general, Java, some C#/.NET, clean applications and libraries design, applications and services architecture, well-designed systems keeping them clean as much as possible.
I helped people at Stack Overflow and occasionally I contribute to open source projects.

Android Maven Plugin
====================

[site](http://simpligility.github.io/android-maven-plugin/),
[source code](https://github.com/simpligility/android-maven-plugin)

* [ACCEPTED] [Added `skipDependencies` option to `ApkMojo` and `DexMojo`](https://github.com/simpligility/android-maven-plugin/pull/632) to be used in cooperation with the [maven-dependency-plugin](https://maven.apache.org/plugins/maven-dependency-plugin/) and [retrolambda-maven-plugin](https://github.com/orfjackal/retrolambda)). Released in [4.4.1](https://simpligility.github.io/android-maven-plugin/changelog.html#4_4_1_released_2016_01_28).
* [ACCEPTED] [Added `unpackOutputDirectory` and `includeNonClassFiles` options](https://github.com/simpligility/android-maven-plugin/pull/779) to the `unpack` goal. Released in [4.6.0](https://simpligility.github.io/android-maven-plugin/changelog.html#4_6_0_released_2019_05_08).

Git
===

[site](https://git-scm.com/),
[source code mirror](https://github.com/git/git)

* [REJECTED][FORKED] The [MediaWiki remote helper](https://www.mediawiki.org/wiki/Git-remote-mediawiki) can store namespace pages in subdirectories. These patches have been accepted to the [Gentoo](https://github.com/gentoo/gentoo/blob/e97050ff8c5fc45e15b899a3f7445594f9daeba9/dev-vcs/git/files/git-2.7.0-mediawiki-subpages.patch) distributive (partially?).
* [REJECTED][FORKED] Indicating git log root marks. See more at the [mailing list](https://public-inbox.org/git/xmqqfu4dzdax.fsf@gitster-ct.c.googlers.com/t/) and [StackOverflow](https://stackoverflow.com/questions/47591021/git-graph-oneline-graph-marker-characters-for-root-commits).

Google Gson
===========

[site and source code](https://github.com/google/gson)

* [ACCEPTED] [Migrated Date JSON (de)serializer to type adapter](https://github.com/google/gson/pull/1070) to make it work at little more efficient. Released in 2.8.1.
* [ACCEPTED][MINOR] [Remove helper methods mentioned in the TODO list](https://github.com/google/gson/pull/1073).
* [ACCEPTED] [Fixed DefaultDateTypeAdapter nullability issue and JSON primitives contract](https://github.com/google/gson/pull/1100). Released in 2.8.2.
* [PENDING] [Support arbitrary Number implementation during deserialization](https://github.com/google/gson/pull/1290)

Maven Enforcer Plugin
=====================

[site](http://maven.apache.org/enforcer/maven-enforcer-plugin/),
[Subversion to Git source code mirror](https://github.com/apache/maven-enforcer)

* [ACCEPTED] Adapted Edward Samson's implementation: [[MENFORCER-247] Add a "require file checksum" rule](https://github.com/apache/maven-enforcer/pull/18) ([MENFORCER-247](https://issues.apache.org/jira/browse/MENFORCER-247), [commit](https://github.com/apache/maven-enforcer/commit/ff9da52d54f5a3a71dd78e98ae64868d1041841b)) to be released soon

RethinkDB
=========

[site](https://rethinkdb.com),
[source code](https://github.com/rethinkdb/rethinkdb)

* [ACCEPTED] With a great support by [Sam Hughes](https://github.com/srh): [Implement bitwise operations](https://github.com/rethinkdb/rethinkdb/pull/6534). Released in [2.4](https://rethinkdb.com/blog/2.4.0-release).

RetroLambda
===========

[site](https://github.com/orfjackal/retrolambda),
[source code](https://github.com/orfjackal/retrolambda)

* [ACCEPTED] [Fixed NullPointerException in ClassInfo accepting java.lang.Object](https://github.com/orfjackal/retrolambda/pull/113). Released in 2.4.0
* [ACCEPTED] With a great support by [Evgeny Mandrikov](https://github.com/Godin): [Added retrolambda.javaHacks configuration option](https://github.com/luontola/retrolambda/pull/143) to work around some `javac` bytecode metadata generation issues. Released in 2.5.5.

Spoilers
========

[site](https://www.mediawiki.org/wiki/Extension:Spoilers),
[source code](https://github.com/lyubomyr-shaydariv/Spoilers)

* [FORKED] Implemented lazy loaded content for Spoilers 1.2.

SpringMVC Router
================

[site and source code](https://github.com/resthub/springmvc-router)

* [STALE] [Remove controller methods visibility restrictions](https://github.com/resthub/springmvc-router/pull/70).

wget
====

[site](https://www.gnu.org/software/wget/),
[source code](https://git.savannah.gnu.org/git/wget.git)

* [PENDING] [Add --custom-html-attrs option to support custom HTML tags and attribute](https://lists.gnu.org/archive/html/bug-wget/2020-01/msg00005.html). See more at [Unix StackExchange](https://unix.stackexchange.com/questions/258835/wget-follow-custom-url-attributes).

VK Dialogue Export
==================

[site and source code](https://github.com/coldmind/vk-dialogue-export.py)

* [ACCEPTED] [Config + Group chats](https://github.com/coldmind/vk-dialogue-export.py/pull/2)
* [PENDING] [Attachments, .auth.ini, forwarded messages, chat type autodetection, internal details fix, chat ID in command line arguments, geo tags, attached photos export, output directories support](https://github.com/coldmind/vk-dialogue-export.py/pull/7)
