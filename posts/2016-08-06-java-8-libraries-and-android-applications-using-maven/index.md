.. title: Java 8 libraries and Android applications using Maven
.. date: 2016-08-06 17:44:00 +0300
.. tags: java, java-8, android, maven, retrolambda
.. category: java
.. type: text

> The original question was originally posted on May 17, 2015 at [StackOverflow](http://stackoverflow.com/questions/30286371/maven-using-java-8-libraries-in-applications-instrumented-with-retrolambda-mave/32510895)

> This article was originally written in Russian and published on September 15, 2015 at [Habrahabr](https://habrahabr.ru/post/266881/)

Java 8 was released in early 2014 featuring some pretty convenient new [features](http://www.oracle.com/technetwork/java/javase/8-whats-new-2157071.html) to simplify trivial coding and letting the developers to simplify their lives.
Some of them are lambda expressions, method and constructor references, interface default methods as the Java language and JVM extensions, and the [Stream API](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) for JDK.
Unfortunately, slow introduction of such new features has a pretty negative impact on another Java-oriented platforms.
GWT and Android still lack the Java 8 language features official support.
However, the last spring GWT 2.8.0 SNAPSHOT versions have lambda expressions support already.
Android things are still different since lambda expressions rely on the Android Java runtime, and not just the compiler.
But Maven 8 lets to solve the Java 8 use problem relatively easy.

<!-- TEASER_END -->

So it happened that all my codebase is based on Maven because:

* it's there for all the time regardless the cumbersome `pom.xml`;
* I can tune the build in one place for all modules regardless the nesting level;
* I can use the single tool to build all my universe modules.

The general purpose libraries from the repository universe are written so they can be used as dependencies for Java SE, GWT and Android applications
As Android still does not support Java 8, these libraries remain not migrated, in Java 6 or 7, as well as the Android applications are.
Nevertheless, working successfully with lambda expressions in the GWT applications, I would like to migrate my entire codebase to Java 8.
Compliling and install these libraries to a local repository is straight-forward:

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-compiler-plugin</artifactId>
	<configuration>
		<source>1.8</source>
		<target>1.8</target>
	</configuration>
</plugin>
```

Once the libraries are installed to the local repository, one might try to build an application.
However, `dex`-ing causes the following error:

```
[INFO] UNEXPECTED TOP-LEVEL EXCEPTION:
[INFO] com.android.dx.cf.iface.ParseException: bad class file magic (cafebabe) or version (0034.0000)
[INFO]  at com.android.dx.cf.direct.DirectClassFile.parse0(DirectClassFile.java:472)
[INFO]  at com.android.dx.cf.direct.DirectClassFile.parse(DirectClassFile.java:406)
[INFO]  at com.android.dx.cf.direct.DirectClassFile.parseToInterfacesIfNecessary(DirectClassFile.java:388)
[INFO]  at com.android.dx.cf.direct.DirectClassFile.getMagic(DirectClassFile.java:251)
[INFO]  at com.android.dx.command.dexer.Main.processClass(Main.java:665)
[INFO]  at com.android.dx.command.dexer.Main.processFileBytes(Main.java:634)
[INFO]  at com.android.dx.command.dexer.Main.access$600(Main.java:78)
[INFO]  at com.android.dx.command.dexer.Main$1.processFileBytes(Main.java:572)
[INFO]  at com.android.dx.cf.direct.ClassPathOpener.processArchive(ClassPathOpener.java:284)
[INFO]  at com.android.dx.cf.direct.ClassPathOpener.processOne(ClassPathOpener.java:166)
[INFO]  at com.android.dx.cf.direct.ClassPathOpener.process(ClassPathOpener.java:144)
[INFO]  at com.android.dx.command.dexer.Main.processOne(Main.java:596)
[INFO]  at com.android.dx.command.dexer.Main.processAllFiles(Main.java:498)
[INFO]  at com.android.dx.command.dexer.Main.runMonoDex(Main.java:264)
[INFO]  at com.android.dx.command.dexer.Main.run(Main.java:230)
[INFO]  at com.android.dx.command.dexer.Main.main(Main.java:199)
[INFO]  at com.android.dx.command.Main.main(Main.java:103)
[INFO] ...while parsing foo/bar/FooBar.class
```

It means that `dx` can't process files compiled with a Java 8 compiler.
So just trying to use [Retrolambda](https://github.com/orfjackal/retrolambda/blob/master/README.md) that, in theory, might fix it:

```xml
<plugin>
	<groupId>net.orfjackal.retrolambda</groupId>
	<artifactId>retrolambda-maven-plugin</artifactId>
	<version>2.0.6</version>
	<executions>
		<execution>
			<phase>process-classes</phase>
			<goals>
				<goal>process-main</goal>
				<goal>process-test</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<defaultMethods>true</defaultMethods>
		<target>1.6</target>
	</configuration>
</plugin>
```

Unfortunately, `foo/bar/FooBar.class` belongs to a library and the error is not eliminated.
`retrolambda-maven-plugin` cannot process the application dependencies in principle because it only can process the classes in the current module (otherwise processing is required directly in the repository).
Simply speaking, the application cannot use Java 8 libraries, but can use Java 8 for the current module only.
It can be fixed in this way:

* unpack all Java 8 dependencies to a directory where the classfiles might be downgraded;
* downgrade the current module bytecode along with the unpacked dependencies bytecode;
* build the DEX and APK files excluding the modules that were unpacked and downgraded.

The current `android-maven-plugin` runs `dx` passing all the dependencies to it making Java 8 bytecode processing impossible.
This is what, roughly speaking, `android-maven-plugin` runs:

```
$JAVA_HOME/jre/bin/java
-Xmx1024M
-jar "$ANDROID_HOME/sdk/build-tools/android-4.4/lib/dx.jar"
--dex
--output=$BUILD_DIRECTORY/classes.dex
$BUILD_DIRECTORY/classes
$M2_REPO/foo1-java8/bar1/0.1-SNAPSHOT/bar1-0.1-SNAPSHOT.jar
$M2_REPO/foo2-java8/bar2/0.1-SNAPSHOT/bar2-0.1-SNAPSHOT.jar
$M2_REPO/foo3-java8/bar3/0.1-SNAPSHOT/bar3-0.1-SNAPSHOT.jar
```

In the example above all Java 8 dependencies are passed to the `dx` tool.
However the plugin itself might feature a dependency filtering options hence to control the dependencies passed to the `dx` tool.
Why it might be useful?
Well, we can assume that some of the dependency binaries are located in a better-to-process place rather than the repository.
Let's say, `${project.build.directory}/classes`.
This is exactly a place where `retrolambda-maven-plugin` can process Java 8 libraries bytecode.

There is a Maven [plugin](https://maven.apache.org/plugins/maven-dependency-plugin/) that can upack the dependencies into a certain directory allowing to process the dependencies the way we want.
For example:

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-dependency-plugin</artifactId>
	<version>2.10</version>
	<executions>
		<execution>
			<phase>process-classes</phase>
			<goals>
				<goal>unpack-dependencies</goal>
			</goals>
			<configuration>
				<includeScope>runtime</includeScope>
				<includeGroupIds>foo1-java8,foo2-java8,foo3-java8</includeGroupIds>
				<outputDirectory>${project.build.directory}/classes</outputDirectory>
			</configuration>
		</execution>
	</executions>
</plugin>
```

I have [implemented](https://github.com/simpligility/android-maven-plugin/pull/632) a few dependency filter management options to an `android-maven-plugin` fork.
They are including and excluding (`excludes` and `includes`) by group ID, arifact ID and version filters.
Artifact IDs and versions specifiers are optional.
All elements identifying an artifact or an artifact group must be separated with a colon.
However it's possible to test Java 8 and Java 8 dependencies in an Android application, while the pull request is still pending to be merged to the original repository.
Just build the plugin fork first:

```bash
# The last upstream merge commit hash
PLUGIN_REVISION=a79e45bc0721bfea97ec139311fe31d959851476

# Clone the fork:
git clone https://github.com/lyubomyr-shaydariv/android-maven-plugin.git

# Make sure to use the expected commit:
cd android-maven-plugin
git checkout $PLUGIN_REVISION

# Building the plugin:
mvn clean package -Dmaven.test.skip=true

# Navigate to the target directory to prepare the fork to be installed to the Maven repository:
cd target
cp android-maven-plugin-4.3.1-SNAPSHOT.jar android-maven-plugin-4.3.1-SNAPSHOT-$PLUGIN_COMMIT.jar

# Fix pom.xml:
cp ../pom.xml pom-$PLUGIN_COMMIT.xml
sed -i "s/<version>4.3.1-SNAPSHOT<\\/version>/<version>4.3.1-SNAPSHOT-$PLUGIN_COMMIT<\\/version>/g" pom-$PLUGIN_COMMIT.xml

# Update the plugin descriptor:
unzip android-maven-plugin-4.3.1-SNAPSHOT-$PLUGIN_COMMIT.jar META-INF/maven/plugin.xml
sed -i "s/<version>4.3.1-SNAPSHOT<\\/version>/<version>4.3.1-SNAPSHOT-$PLUGIN_COMMIT<\\/version>/g" META-INF/maven/plugin.xml
zip android-maven-plugin-4.3.1-SNAPSHOT-$PLUGIN_COMMIT.jar META-INF/maven/plugin.xml

# Install the plugin finally:
mvn org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -DpomFile=pom-$PLUGIN_COMMIT.xml -Dfile=android-maven-plugin-4.3.1-SNAPSHOT-$PLUGIN_COMMIT.jar
```

Once it's done, `pom.xml` might be configured for the application:

```xml
<!-- Enable Java 8 support for the current module -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.2</version>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
    </configuration>
</plugin>

<!-- Unpack the Java 8 dependencies classes to the current build directory -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>2.10</version>
    <executions>
            <execution>
                <phase>process-classes</phase>
                <goals>
                    <goal>unpack-dependencies</goal>
                </goals>
                <configuration>
                    <includeScope>runtime</includeScope>
                    <!-- It's crucial to specify Java 8 dependencies only -->
                    <includeGroupIds>foo1-java8,foo2-java8.foo3-java8</includeGroupIds>
                    <outputDirectory>${project.build.directory}/classes</outputDirectory>
                </configuration>
        </execution>
    </executions>
</plugin>

<!-- Process the bytecode -->
<plugin>
    <groupId>net.orfjackal.retrolambda</groupId>
    <artifactId>retrolambda-maven-plugin</artifactId>
    <version>2.0.6</version>
    <executions>
        <execution>
            <phase>process-classes</phase>
            <goals>
                <goal>process-main</goal>
                <goal>process-test</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <defaultMethods>true</defaultMethods>
        <target>1.6</target>
    </configuration>
</plugin>

<!-- DEX-ing all non-Java 8 dependencies (by that time the target/classes directory already has the dependencies that dx can process) and pack all the stuff to an APK -->
<plugin>
    <groupId>com.simpligility.maven.plugins</groupId>
    <artifactId>android-maven-plugin</artifactId>
    <version>4.3.1-SNAPSHOT-a79e45bc0721bfea97ec139311fe31d959851476</version>
    <executions>
        <execution>
            <phase>package</phase>
        </execution>
    </executions>
    <configuration>
        <androidManifestFile>${project.basedir}/src/main/android/AndroidManifest.xml</androidManifestFile>
        <assetsDirectory>${project.basedir}/src/main/android/assets</assetsDirectory>
        <resourceDirectory>${project.basedir}/src/main/android/res</resourceDirectory>
        <sdk>
            <platform>19</platform>
        </sdk>
        <undeployBeforeDeploy>true</undeployBeforeDeploy>
        <proguard>
            <skip>true</skip>
            <config>${project.basedir}/proguard.conf</config>
        </proguard>
        <excludes>
            <exclude>foo1-java8</exclude>
            <exclude>foo2-java8</exclude>
            <exclude>foo3-java8</exclude>
        </excludes>
    </configuration>
    <extensions>true</extensions>
    <dependencies>
        <dependency>
            <groupId>net.sf.proguard</groupId>
            <artifactId>proguard-base</artifactId>
            <version>5.2.1</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</plugin>
```

That's essentially all.
It's worth noting that this approach only means use of Java 8 language features, and not the standard libraries like Stream API.
Also, this approach can be used not just to make an Android application Java 8 libraries aware, but let it process 3rd party dependencies bytecode in any way.
However, I wouldn't say that I really like this approach very much.

Maybe it's all much simpler for another build tools.
I don't even know if it can be simpler in Maven and if it's not re-inventing a wheel, but nevertheless it was fun to make Maven do what I want it to do.

----

> This change was [merged](https://github.com/simpligility/android-maven-plugin/commit/e56d8daa65a61b1d16f12819500a3aabcb32ba89) to the original repository and the feature is available since Android Maven Plugin 4.4.1.
