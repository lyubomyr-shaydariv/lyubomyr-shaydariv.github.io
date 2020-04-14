.. title: Ukrainian Fillwords simple crack
.. date: 2016-11-22 22:02:00 +0300
.. tags: cracking, android, games
.. category: cracking
.. type: text

> Dedicated to Anna

A few days ago my girlfriend got a new mobile device.
In order to migrate all of the stuff from the old device to the new one, I decided to use a good old [`adb`]({% post_url 2016-08-27-when-android-backup-and-restore-tools-may-fail %}).
While spending a few hours of talking to each other, the `adb` tool was backing up and restoring the necessary application and data.
However, there was a simple game, [Ukrainian Fillwords (Філворди Українською)](https://play.google.com/store/apps/details?id=com.games.admo.fillwords), I couldn't migrate.
Well, let's see what was under the hood.

<!-- TEASER_END -->

I mean, I couldn't migrate my girlfriend's game achievements to the new device.
I had some free time to dig it deeper and find out the root cause.

To start the experiment, I made a backup of the game using `adb` first:

```
adb backup -f com.games.admo.fillwords.ab -apk com.games.admo.fillwords
```

Unfortunately, the Android backup format is not that easy to use, so it must be converted to be processible with other tools.
There is a great tool, [nelenkov/android-backup-extractor](https://github.com/nelenkov/android-backup-extractor), written by [Nikolay Elenkov](https://github.com/nelenkov), allowing to convert Android backups to `tar` files.
I don't really know what the tool does behind the scenes, but it does the job perfectly.
The precompiled binary is still [here](/downloads/abe-all-fcb4ee5af3ec973a406709de889a5cf340b61ce1.jar).
So now an easy to read `tar` file can be converted from the backup:

```
java -jar abe-all.jar unpack com.games.admo.fillwords.ab com.games.admo.fillwords.ab.tar
```

Now let's extract the game application APK.
I also don't really know what could be an exact name of the `apk` file, but it was `com.games.admo.fillwords-1.apk` to me:

```
tar -xvf com.games.admo.fillwords.ab.tar apps/com.games.admo.fillwords/a/com.games.admo.fillwords-1.apk \
	&& mv apps/com.games.admo.fillwords/a/com.games.admo.fillwords-1.apk . \
	&& rm -rf apps
```

Then it's necessary to extract the classes `dex` file where the application classes are stored:

```
unzip -p com.games.admo.fillwords-1.apk classes.dex > classes.dex
```

... and convert the `dex` file to a regular `jar` using [dex2jar](https://github.com/pxb1988/dex2jar):

```
java -cp d2j-base-cmd-2.0.jar:dex-translator-2.0.jar:dex-tools-2.0.jar:dex-reader-2.0.jar:dex-ir-2.0.jar:dex-reader-api-2.0.jar:asm-debug-all-4.1.jar com.googlecode.dex2jar.tools.Dex2jarCmd classes.dex -o classes.dex.jar
```

... and then decompile the `jar` and save the decompiled Java source code using [fernflower](https://github.com/fesh0r/fernflower).

```
mkdir -p out \
	&& java -jar fernflower.jar classes.dex.jar out \
	&& mv out/classes.dex.jar classes.dex.jar.src.zip \
	&& rm -rf out
```

Well...
Fortunately, the classes are neither obfuscated nor protected in any other way.
After spending a few minutes of quick investigation, I've found out that there's a special class holding the game state.
Unfortunately `fernflower` cannot decompile the most interesting code, and I spent more time to spot the game state file target.
The problematic class `com.games.admo.fillwords.Settings` and its `public static void save(FileIO param0)` method cannot be decompiled.
However, viewing the class file (not the decompiled Java source code) lets you quickly spot the target file: it's just named `.fillwords`, and I remember the game stores the file using a special game state manager class that's bound to the SD card.
Then I ran the game in order to complete the first level to let the game state file be written, and ran the following command:

```
adb pull /sdcard/.fillwords
```

Yep, this is what it does, and the file is just in the root of the SD card.
The game state is now in my hands.
Note that files starting with a dot (`.`) are considered hidden in Unix-like systems, so merely running `ls` (`ll`) won't show you such files by default, whilst `ls -a` will.

Let's see file contents with enumerated lines produced by `cat .fillwords | nl`:

```
     1  2
     2  true
     3  24
     4  22
     5  2
     6  3
     7  -1
     8  3
     9  0
    10  0
    11  0
```

Pretty simple, right?
But it's just text, and it needs to be dissected.
I'll get it powered up, I believe.

[JD-GUI](https://github.com/java-decompiler/jd-gui) can still show the byte code for `com.games.admo.fillwords.Settings#save(File)`.
I don't know if Fernflower can do the same, but sometimes I prefer JD-GUI, and now the latter helper even more.
I didn't have any Java disassembling experience before, but let it be the very first time.
I may be wrong with my "in-head" disassembling, but I hope the general line is correct, and the game state file algorithm can be restored easily.
By analyzing the disassembled code, the following mappings are revealed:

Line #1 gives `2` (the file begin marker) and a new line.
This is a file magic number, and this value is constant by definition.

```
//   27: aload_0
//   28: iconst_2
//   29: invokestatic 127	java/lang/Short:toString	(S)Ljava/lang/String;
//   32: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   35: aload_0
//   36: ldc -123
//   38: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Line #2 gives `true` for `soundEnabled` and a new line.
Yes, I didn't turn off the sound, but I usually don't even turn on the device sound.
It doesn't really make much sense to me, but it's written and must be considered.

```
//   41: aload_0
//   42: getstatic 26	com/games/admo/fillwords/Settings:soundEnabled	Z
//   45: invokestatic 136	java/lang/Boolean:toString	(Z)Ljava/lang/String;
//   48: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   51: aload_0
//   52: ldc -123
//   54: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Line #3 gives `24` for `score` and a new line.
A not very interesting value to change, since I believe achievements are more fun (a game should be played, right?).

```
//   57: aload_0
//   58: getstatic 28	com/games/admo/fillwords/Settings:score	I
//   61: invokestatic 139	java/lang/Integer:toString	(I)Ljava/lang/String;
//   64: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   67: aload_0
//   68: ldc -123
//   70: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Line #4 gives `22` for `tipNumber` and a new line.
This is how many available hints you have to open next letter squares in the game.
And it's probably the most desired game state value for a cheater.
Note that setting this value a big enought value, like a few thousands, lets you to walk through the game by just pressing the hint button. :)
You decide.

```
//   73: aload_0
//   74: getstatic 30	com/games/admo/fillwords/Settings:tipNumber	I
//   77: invokestatic 139	java/lang/Integer:toString	(I)Ljava/lang/String;
//   80: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   83: aload_0
//   84: ldc -123
//   86: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Line #5 gives `2` for `maxSection` (available in the loop below) and a new line.
All levels of the game are grouped in six sections.
By default the first two level sections are open, so this is why `2` is written here.

```
//   89: aload_0
//   90: getstatic 32	com/games/admo/fillwords/Settings:maxSection	I
//   93: invokestatic 139	java/lang/Integer:toString	(I)Ljava/lang/String;
//   96: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   99: aload_0
//   100: ldc -123
//   102: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Line #6 gives `3` for `sectionInfo` (not sure what it is; for each section) and a new line.
This is actually a loop begin to iterate over all game level sections (6 in total) and a check to break the iteration loop.

```
//   105: iconst_0
//   106: istore_1
//   107: iload_1
//   108: getstatic 32	com/games/admo/fillwords/Settings:maxSection	I
//   111: if_icmpge +64 -> 175
//   114: aload_0
//   115: getstatic 79	com/games/admo/fillwords/Settings:sectionInfo	[I
//   118: iload_1
//   119: iaload
//   120: invokestatic 139	java/lang/Integer:toString	(I)Ljava/lang/String;
//   123: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   126: aload_0
//   127: ldc -123
//   129: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Line #7 gives `-1` for `tipIndex` (for each section) and a new line.
Honestly, have no idea what it is, but still don't really care.

```
//   132: aload_0
//   133: getstatic 81	com/games/admo/fillwords/Settings:tipIndex	[S
//   136: iload_1
//   137: saload
//   138: invokestatic 127	java/lang/Short:toString	(S)Ljava/lang/String;
//   141: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   144: aload_0
//   145: ldc -123
//   147: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Line #8 gives `3` for `maxLevel` (for each section) and a new line.
This is where I could really cheat to open levels I have never reached to.

```
//   150: aload_0
//   151: getstatic 83	com/games/admo/fillwords/Settings:maxLevel	[S
//   154: iload_1
//   155: saload
//   156: invokestatic 127	java/lang/Short:toString	(S)Ljava/lang/String;
//   159: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
//   162: aload_0
//   163: ldc -123
//   165: invokevirtual 131	java/io/BufferedWriter:write	(Ljava/lang/String;)V
```

Lines #9, #10, and #11 are generated in a loop (repeat 3 steps above, if there is more than 1 section):

```
//   168: iload_1
//   169: iconst_1
//   170: iadd
//   171: istore_1
//   172: goto -65 -> 107
//   175: aload_0
```

After a brief analysis of an extremely simple game state writer, I can make a simple map of the overall game state file structure:

```
2
<soundEnabled>
<score>
<tipNumber>
<maxSection>
<FOR EACH SECTION>
	<sectionInfo>
	<tipIndex>
	<maxLevel>
</FOR EACH SECTION>
```

That's all.

And yes, we didn't cheat while and after the migration of the `/sdcard/.fillwords` from the old device to the new one. :)
