# [Quick Start] Building boon for the QBit microservice engine

##Overview
Boon is not only a JSON parsing project. It is more than that, but JSON parsing is intrinsic to what Boon is all about. Boon is probably the fastest way to serialize and parse JSON in Java so far for your project. This parser was forked/merged into Groovy 2.3. The Boon and Groovy 2.3 JSON parsers are a lot faster than mainstream JSON parsers. Boon JSON is FAST! In addition it has a very easy to use, convention-based API. some of the things you can do with Boon:


-Boon comes with helper methods that allow you to easily create lists, sets, maps, concurrent maps, sorted maps, sorted sets, etc. ( safeList, list, set, sortedSet, safeSet, safeSortedSet, etc...)

-With boon You can index maps, lists, arrays, etc. using the idx operator.

-Boon has the concept of universal operators similar to Python like len.

-Boon can read in an entire file in one line of code.

-And much more for more details please visit the articles located at the end of this wiki.



##Building Boon

This wiki will walk you through the process of building this very powerful java tool `Boon`

##what you will Build

You will build Boon on your machine using [maven. ](http://maven.apache.org/)

##How to complete this guide

In order to complete this guide successfully you will need the following installed on your machine:

- Maven; if you need help installing it, visit [Installing Maven. ](http://maven.apache.org/download.cgi#Installation)
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.

Now that your machine is all ready let's get started:

- [Download Boon ](https://github.com/fadihub/boon/archive/master.zip) and unzip the source repository for this guide, or clone it using Git with your terminal:

```bash
git clone https://github.com/fadihub/boon.git
```

cd into `~/boon` then run the following:

```bash
mvn clean install
```
This will take a minute especially if you are using maven for the first time.

####pom.xml Linsting
The pom.xml file is the core of a project's configuration in Maven. It is a single configuration file that contains the majority of information required to build a project in just the way you want.


`~/boon/pom.xml/`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.sonatype.oss</groupId>
        <artifactId>oss-parent</artifactId>
        <version>5</version>
    </parent>

    <groupId>io.fastjson</groupId>
    <artifactId>boon-bundle</artifactId>
    <version>0.31-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>Boon and subprojects</name>

    <description>Boon Bundle</description>
    <url>http://richardhightower.github.io/site/Boon</url>

    <licenses>
        <license>
            <name>The Apache Software License, Version 2.0</name>
            <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
            <distribution>repo</distribution>
        </license>
    </licenses>

    <scm>
        <connection>scm:git:https://github.com/boonproject/boon.git</connection>
        <developerConnection>scm:git:https://github.com/boonproject/boon.git</developerConnection>
        <url>https://github.com/boonproject/boon</url>
    </scm>

    <developers>
        <developer>
            <id>RichardHightower</id>
            <name>Richard Hightower</name>
            <url>https://github.com/RichardHightower</url>
        </developer>
    </developers>

    <modules>
        <module>boon</module>
        <module>bnsf</module>
        <module>etcd</module>
    </modules>

    <properties>
        <maven.compiler.source>1.7</maven.compiler.source>
        <maven.compiler.target>1.7</maven.compiler.target>
        <groovy.version>2.3.2</groovy.version>
        <junit.version>4.11</junit.version>
        <guava.version>18.0</guava.version>
        <vertx.version>2.1.2</vertx.version>
        <vertx-testtools.version>2.0.3-final</vertx-testtools.version>
        <hamcrest.version>1.3</hamcrest.version>
        <kryo.version>2.23.0</kryo.version>
        <leveldb.version>0.7</leveldb.version>
        <leveldbjni.version>1.8</leveldbjni.version>
        <mongo-java-driver.version>2.12.0</mongo-java-driver.version>
        <mysql-connector-java.version>5.1.30</mysql-connector-java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- internal -->
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>boon</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>qbit</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>slumberdb-core</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>slumberdb-service-model-common</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>slumberdb-mysql</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>slumberdb-leveldb</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>slumberdb-service-client</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.fastjson</groupId>
                <artifactId>slumberdb-service-model</artifactId>
                <version>${project.version}</version>
            </dependency>

            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>${junit.version}</version>
            </dependency>
            <dependency>
                <groupId>org.codehaus.groovy</groupId>
                <artifactId>groovy</artifactId>
                <version>${groovy.version}</version>
            </dependency>
            <dependency>
                <groupId>com.google.guava</groupId>
                <artifactId>guava</artifactId>
                <version>${guava.version}</version>
            </dependency>
            <dependency>
              <groupId>io.vertx</groupId>
              <artifactId>vertx-core</artifactId>
              <version>${vertx.version}</version>
            </dependency>
            <dependency>
              <groupId>io.vertx</groupId>
              <artifactId>vertx-platform</artifactId>
              <version>${vertx.version}</version>
            </dependency>
            <dependency>
              <groupId>io.vertx</groupId>
              <artifactId>vertx-hazelcast</artifactId>
              <version>${vertx.version}</version>
            </dependency>
            <dependency>
              <groupId>io.vertx</groupId>
              <artifactId>testtools</artifactId>
              <version>${vertx-testtools.version}</version>
            </dependency>
            <dependency>
                <groupId>org.hamcrest</groupId>
                <artifactId>hamcrest-all</artifactId>
                <version>${hamcrest.version}</version>
            </dependency>
            <dependency>
                <groupId>com.esotericsoftware.kryo</groupId>
                <artifactId>kryo</artifactId>
                <version>${kryo.version}</version>
            </dependency>
            <dependency>
                <groupId>org.iq80.leveldb</groupId>
                <artifactId>leveldb</artifactId>
                <version>${leveldb.version}</version>
            </dependency>
            <dependency>
                <groupId>org.fusesource.leveldbjni</groupId>
                <artifactId>leveldbjni-all</artifactId>
                <version>${leveldbjni.version}</version>
            </dependency>
            <dependency>
                <groupId>org.mongodb</groupId>
                <artifactId>mongo-java-driver</artifactId>
                <version>${mongo-java-driver.version}</version>
            </dependency>
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>${mysql-connector-java.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-source-plugin</artifactId>
                    <version>2.1.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-javadoc-plugin</artifactId>
                    <version>2.9.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>2.17</version>
                </plugin>
                <plugin>
                    <artifactId>maven-surefire-report-plugin</artifactId>
                    <version>2.17</version>
                </plugin>
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>2.6</version>
                </plugin>
                <plugin>
                    <artifactId>maven.failsafe.plugin.version</artifactId>
                    <version>2.14</version>
                </plugin>
                <plugin>
                    <artifactId>maven-dependency-plugin</artifactId>
                    <version>2.7</version>
                </plugin>
                <plugin>
                    <groupId>io.vertx</groupId>
                    <artifactId>vertx-maven-plugin</artifactId>
                    <version>2.0.8-final</version>
                    <!--
                    You can specify extra config to the plugin as required here
                    <configuration>
                       <configFile>/path/to/MyVerticle.conf</configFile>
                       <instances>1</instances>
                       <classpath>src/main/resources/:src/test/resources/:target/classes/:target/test-classes/</classpath>
                    </configuration>
                    -->
                    <!-- Make sure that the plugin uses the vert.x versions from this pom.xml not from its own pom.xml -->
                    <dependencies>
                        <dependency>
                            <groupId>io.vertx</groupId>
                            <artifactId>vertx-platform</artifactId>
                            <version>${vertx.version}</version>
                        </dependency>
                        <dependency>
                            <groupId>io.vertx</groupId>
                            <artifactId>vertx-core</artifactId>
                            <version>${vertx.version}</version>
                        </dependency>
                        <dependency>
                            <groupId>io.vertx</groupId>
                            <artifactId>vertx-hazelcast</artifactId>
                            <version>${vertx.version}</version>
                        </dependency>
                    </dependencies>
                </plugin>
            </plugins>
        </pluginManagement>

        <plugins>
            <plugin>
                <artifactId>maven-source-plugin</artifactId>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <reporting>
        <plugins>
            <plugin>
                <artifactId>maven-javadoc-plugin</artifactId>
            </plugin>
        </plugins>
    </reporting>

    <profiles>
        <profile>
            <id>Windows</id>
            <activation>
                <os>
                    <family>Windows</family>
                    <arch>x86</arch>
                </os>
            </activation>
            <properties>
                <skipTests>true</skipTests>
            </properties>
        </profile>
    </profiles>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Congrats now you have `Boon` installed on your machine.

##Why use Boon?
Easily read in files into lines or a giant string with one method call.
Works with files, URLs, class-path, etc. Boon IO support will surprise you how easy it is.
Boon has Slice notation for dealing with Strings, Lists, primitive arrays, Tree Maps, etc.
If you are from Groovy land, Ruby land, Python land, and you have to use
Java then Boon might give you some relief from API bloat.
Boon lets Java be Java, but adds the missing productive APIs from Python, Ruby, and Groovy.


##Related Articles
* [Boon Home](https://github.com/RichardHightower/boon/wiki)
* [Java Boon Byte Buffer Builder](https://github.com/RichardHightower/boon/wiki/Boon's-Byte-Buffer-Builder)
* [Java Boon Slice Notation](https://github.com/RichardHightower/boon/wiki/Boon-Slice-Notation)
* [Java Boon Slice's work with TreeSets](https://github.com/RichardHightower/boon/wiki/Sets-and-Slice-Notation-for-Java-Boon!)
* [Java Boon Description](https://github.com/RichardHightower/boon/wiki)
* [Boon Source](https://github.com/RichardHightower/boon/wiki)
* [Introducing Boon October 2013](http://rick-hightower.blogspot.com/2013/10/introducing-boon-for-java.html)
* [Java Slice Notation](http://rick-hightower.blogspot.com/2013/10/java-slice-notation-to-split-up-strings.html)
* [What if Java collections were easy to search and sort?](http://rick-hightower.blogspot.com/2013/11/what-if-java-collections-and-java.html)
* [Boon HTTP utils](http://rick-hightower.blogspot.com/2013/11/stackoverflow-question-on-posting-http.html)
* [Boon Java JSON parser Benchmarks or hell yeah JSON parsing is damn fast!](http://rick-hightower.blogspot.com/2013/11/benchmark-for-json-parsing-boon-scores.html)
* [Boon JSON parser is really damn fast! Part II](http://rick-hightower.blogspot.com/2013/12/boon-fastest-way-to-turn-json-into.html)
* [Boon JSON parser Round III now just not fast as but much faster than other Java JSON parsers](http://rick-hightower.blogspot.com/2013/12/here-we-go-again-latest-round-of.html)
* [Boon World's fastest Java JSON parser Round IV from fast to blazing to rocket fuel aka Braggers going to brag](http://rick-hightower.blogspot.com/2013/12/worlds-fastest-json-parser.html)
* [Boon gets adopted by JSON Path as the default Java JSON parser](http://rick-hightower.blogspot.com/2013/12/jsonpath-decides-boon-is-fastest-way-to.html)
* [Boon graphics showing just how fast Boon JSON parsing is - about 50% to 200% faster than the graphs shown here now so wicked fast became wickeder - just got sick of making graphics](http://rick-hightower.blogspot.com/2013/12/boon-json-parser-seems-to-be-fastest.html)
* [10 minute guide to Boon JSON parsing after I added @JsonIgnore, @JsonProperty, @JsonView, @Exposes, etc.](http://rick-hightower.blogspot.com/2014/01/boon-json-in-five-minutes-faster-json.html)
* [Hightower speaks to the master of Java JSON parsing, the king of speed The COW TOWN CODER!](http://rick-hightower.blogspot.com/2014/01/boon-jackson-discussion-between.html)
* [Boon provides easy Java objects from lists, from maps and from JSON.](http://rick-hightower.blogspot.com/2014/02/boon-fromlist-frommap-and-fromjson.html)
