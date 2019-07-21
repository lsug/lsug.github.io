---
layout: post
title:  "Workshop: Run Scalac, run!"
date:   2019-07-17 19:04:06 +0100
author: Zainab Ali
categories: jekyll update
---

# Prerequisites

## Sign up!

- Sign up to the [Meetup](https://www.meetup.com/london-scala/events/260846887/?gj=co2&rv=co2&_xtd=gatlbWFpbF9jbGlja9oAJDQwNzM3YjljLTdmMTQtNDRjMi04NjI2LThhNDdiODZkNmZkNw).
- Sign up to [SkillsMatter](https://skillsmatter.com/meetups/12565-workshop-improving-compile-times) so you can enter the building
- Join the [Gitter channel](https://gitter.im/lsug/23-07-19-workshop)

## Tools

This workshop involves an unwieldy amount of tools.

Please come with the following tools installed, as we won't have time to set these up within the workshop.

You will make your life much, much easier if you install these tools before the workshop.  Rory Graves has kindly put together some [setup instructions](https://docs.google.com/document/d/1-Tq3wpqHgH8XIIM2noolVZWs2ENSOZhccS-TkFSdNIs/edit#).  If you have any problems, please ask on the Gitter channel.  We'll be very happy to help.

- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Java 8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- [sbt 1.2.8](https://www.scala-sbt.org/release/docs/Setup.html)
- [Scala 2.12.8 and 2.13.0](https://www.scala-lang.org/download/)
- [Zipkin](https://zipkin.io/pages/quickstart.html) - either the docker image, jar or source code
- [graphviz](https://www.graphviz.org/download/)
- [bloop](https://scalacenter.github.io/bloop/setup)
- [GraalVM](https://www.graalvm.org/downloads/) - either the Community or Enterprise edition
- [clone my fork of the Akka repository](https://github.com/zainab-ali/akka)
- [clone the Bloop GitHub repository](https://github.com/scalacenter/bloop)
- [clone the Scalac Profiling repository](https://github.com/scalacenter/scalac-profiling)

Start the bloop server beforehand, as this will download more dependencies.

### ArchLinux Setup

1. Install the following tools with `pacman`:

   ```console
   lsug$ pacman -S git jdk8-openjdk sbt graphviz
   ```

2. Install the following from the [AUR](https://wiki.archlinux.org/index.php/Arch_User_Repository) with your favourite tool (I use `auracle`)
   - bloop
   - graal-bin

3. Create an `lsug` directory and clone my fork of `akka`.  Update `akka-stream-typed`.  This will download all of its dependencies.

   ```console
   lsug$ git clone https://github.com/zainab-ali/akka.git --branch workshop-start --single-branch --depth=1
   lsug$ cd akka
   lsug$ sbt
   akka> akka-stream-typed/update
   ```

#### Java 8

Check that you have Java 8 and graal installed

```console
lsug$ archlinux-java status
Available Java environments:
  java-8-openjdk (default)
  java-8-graal
```

You can switch between java versions with the `archlinux-java set` command.

#### Zipkin

If you're using docker, start the `openzipkin/zipkin` container

```console
lsug$ docker
lsug$ docker run -d -p 9411:9411 --name zipkin openzipkin/zipkin
lsug$ docker stop zipkin
```

Alternatively, download the zipkin jar

```console
lsug$ mkdir zipkin
lsug$ cd zipkin
zipkin$ curl -sSL https://zipkin.io/quickstart.sh | bash -s
zipkin$ ls
zipkin.jar
```

#### Bloop repository

Clone bloop and update its submodules

```console
lsug$ git clone https://github.com/scalacenter/bloop.git --single-branch --depth=1
lsug$ cd bloop
bloop$
bloop$ git submodule update --init
```

#### Scalac-profiling repository


Clone scalac-profiling and update its submodules

```console
lsug$ git clone https://github.com/scalacenter/scalac-profiling.git --single-branch --depth=1
lsug$ cd scalac-profiling
scalac-profiling$
scalac-profiling$ git submodule update --init
```

### MacOS Setup

These instructions are courtesy of Rory Graves.  Thank you Rory!

The easiest way to install packages on Mac is via homebrew

#### Homebrew

https://brew.sh/

Run:

```console
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

#### XCode

Run:
```console
$ xcode-select --install
```

#### Java8

```console
$ brew tap caskroom/versions brew cask install java8
```

#### Graphviz

```
$ brew install graphviz
```

#### SBT  1:

```console
$ brew install sbt@1
```

Other SBT options can be found on the SBT [Setup guide](https://www.scala-sbt.org/release/docs/Setup.html)

#### git

```console
$ brew install git
```

#### Zipkin

See the Zipkin instructions for ArchLinux.

#### Graal

https://www.graalvm.org/downloads/

You can download the CE version from Github or the EE version from OTN - n.b. The EE one is faster, but you need a license for commercial use.

#### Bloop repository

See the bloop repository instructions for ArchLinux.

#### Scalac-Profiling repository

See the scalac-profiling repository instructions for ArchLinux.

# Workshop contents

For this workshop, we'll be using [akka](https://github.com/akka/akka).  The akka codebase is in pretty good condition, so we'll use an older version to make improvements on.

1. Clone the `workshop-start` branch of **my fork** of akka

   ```console
   lsug$ git clone https://github.com/zainab-ali/akka.git --branch workshop-start --single-branch --depth=1
   lsug$ cd akka
   akka$
   ```

# Profiling

## Bloop

Bloop is a build server we can use to inspect the akka build.

1. Follow the [Bloop Installation Guide](https://scalacenter.github.io/bloop/setup) to install bloop.

2. The the file `akka/project/bloop.sbt` adds the bloop plugin

   ```scala
   addSbtPlugin("ch.epfl.scala" % "sbt-bloop" % "1.3.2")
   ```

2. Open an sbt shell in the `akka` directory

   ```console
   akka$ sbt
   akka>
   ```

   Keep the sbt shell open for the workshop.

3. Export the build

   ```console
   akka> bloopInstall
   ```

You should now be able to use `bloop` to compile your project.

Check that you can call bloop

```console
akka$ bloop --help

bloop 1.3.2
Usage: bloop [options] [command] [command-options]

Available commands: about, autocomplete, bsp, clean, compile, configure, console, help, link, projects, run, test
```

Try listing akka's projects:

```console
akka$ bloop projects

akka
akka-actor
akka-actor-test
akka-actor-testkit-typed
...
```

### Viewing build traces

Bloop outputs build traces.  These can be viewed with [Zipkin](https://zipkin.io/).

1. Pull and start Zipkin.

   If you have docker, this can be done with:

   ```console
   lsug$ docker run -d -p 9411:9411 openzipkin/zipkin
   ```
   If you have the Zipkin jar in a `zipkin` directory, this can be done with

   ```
   zipkin$ java -jar zipkin.jar
   ```

2. Compile the `akka-stream-typed` project

   ```console
   akka$ bloop compile akka-stream-typed
   ```

   Bloop should detect Zipkin automatically and emit traces to it.

   You should be able to see Zipkin traces at [http://localhost:9411/zipkin/](http://localhost:9411/zipkin/)

Let's compile `akka-stream-typed` a few more times.

3. Run the `warmup.sh` script with the project `akka-stream-typed`

   ```console
   akka$ ./warmup.sh akka-stream-typed
   ```

Take a look at the Zipkin traces.  Bloop uses the same JVM instance each time it compiles the project.  Notice that the compilation times get shorter and shorter as the JVM warms up.

### Examining a trace

Each akka project has a `scalac` span.  Some of these run sequentially, but some run in parallel.
The order in which the spans appear depends on the build graph.

Take a look at akka's build graph.

```console
akka$ bloop projects --dot-graph > build-graph.dot
akka$ dot -o build-graph.svg -Tsvg build-graph.dot
```

The `dot` tool is installed with [Graphviz](https://www.graphviz.org/download/).

Open the `akka/build-graph.svg` file using your browser.

## Build pipelining

Projects only depend on the typer phase of their dependency projects.  It's possible to start compilation of a project after its dependency projects have finished the typer phase, but before they've finished compiling completely.  This is termed *build pipelining*.  Bloop supports this with the `--pipeline` option.  Run `bloop compile` with the `--pipeline` option.

```console
akka$ bloop clean
akka$ bloop compile --pipeline akka-stream-typed
```

Sort the Zipkin traces by newest first to see the trace with pipelining.  You should notice that some `scalac` spans start before the previous ones have finished.

# Benchmarking

By using pipelining, we should have sped up our build.  We can only really tell this by benchmarking.

Benchmarking requires an environment with consistent CPU performance (it doesn't need to be good, just consistent).  It needs some careful setup.

## Setting up your environment

To achieve a consistent environment:
 - stop all background processes
 - stop any scheduled processes (e.g. by disabling cron)
 - disable any CPU optimizations
 - set the CPU frequency to a fixed value

Because benchmarking is highly dependent on the CPU frequency, benchmarking from within a VM or container will lead to unreliable results.
It's unlikely that you'll be able to do this within the timespan of the workshop, however I can walk you through the steps required for my own lapytop.

1. Stop any intensive `systemctl` processes
2. Disable hyperthreading
3. Set the CPU frequency

## Setting up the benchmarking suite

Even if you can't setup your environment, it's worth experimenting with bloop's benchmarking suite.
Bloop contains a benchmarking suite based on JMH.

1. Checkout bloop

   ```console
   lsug$ git clone https://github.com/scalacenter/bloop.git --single-branch --depth=1
   lsug$ cd bloop
   bloop$
   bloop$ git submodule update --init
   ```

2. Delete all lines in `bloop/buildpress/src/main/resources/bloop-community-build.buildpress`.

3. Add an entry called `start` for the `workshop-start` akka branch:

   ```
   start,https://github.com/zainab-ali/akka.git#workshop-start
   ```

4. Enter `sbt` and run `exportCommunityBuild`

   ```console
   bloop$ sbt
   bloop> exportCommunityBuild
   ```

## Running the benchmarks

6. In the sbt shell run

   ```console
   bloop> benchmarks/jmh:run .*HotBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=start -p projectName=akka-stream-typed
   ```

   This runs 7 warm up iterations and 5 main iterations.  Once completed, you should be given a score for each iteration.
   Make a note of your scores.

6. Bloop contains a `HotPipelinedBloopBenchmark` class for benchmarking pipelining

   ```console
   bloop> benchmarks/jmh:run .*HotPipelinedBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=start -p projectName=akka-stream-typed
   ```
   Do you notice a decrease in score?

# Upgrades

## Scala

Significant improvements to `scalac` are made in each new scala version. You can see the decrease in compile times in the [Grafana dashboard](https://scala-ci.typesafe.com/grafana/dashboard/db/scala-benchmark).

The akka `workshop-scala-2.13` branch contains a Scala 2.13 upgrade.

1. Add another entry to `bloop/buildpress/src/main/resources/bloop-community-build.buildpress` called `scala213` for the `workshop-scala-2.13` branch.

   ```
   scala213,https://github.com/zainab-ali/akka.git#workshop-scala-2.13
   ```

2. Export the community build once again

   ```console
   bloop> exportCommunityBuild
   ```

3. Run the benchmarks for the Scala 2.13
   ```console
   bloop> benchmarks/jmh:run .*HotPipelinedBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=scala213 -p projectName=akka-stream-typed
   ```

   Do you see a decrease in score?

## Graal VM

The scala compiler runs best on the Java 8 Graal VM.

1. Download and install the [GraalVM](https://www.graalvm.org/downloads/)
2. Exit the sbt shell for the bloop project

   ```console
   bloop> exit
   bloop$
   ```
3. Switch your JVM to Graal.

   - For MacOS, follow [this guide](https://blog.softwaremill.com/graalvm-installation-and-setup-on-macos-294dd1d23ca2).
   - For ArchLinux, use `pacman` and switch java versions with `archlinux-java`

4. Check that you're using the Graal VM

   ```console
   bloop$ java -version
   openjdk version "1.8.0_..."
   OpenJDK Runtime Environment (build ...)
   OpenJDK 64-Bit GraalVM CE 19.1.0 (build ...)
   ```

4. Enter the sbt shell and run the benchmarks once again
   ```console
   $ sbt
   bloop> benchmarks/jmh:run .*HotPipelinedBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=scala213 -p projectName=akka-stream-typed
   ```

   Do you see a decrease in score?

# Compile less code

scalac will be faster if it has less code to compile.  You can
 - use [Scalafix](https://github.com/scalacenter/scalafix) to delete unused imports and local variables
 - keep an eye on [ScalaClean](https://github.com/rorygraves/ScalaClean), a project for detecting dead code

# Profiling

We will now look into profiling `scalac` for the `akka-stream-typed` project.

## -Ystatistics

1. In `akka/project/AkkaBuild.scala`, add the `-Ystatistics` option to the `DefaultScalacOptions`.

   ```scala
   final val DefaultScalacOptions = Seq("-encoding", "UTF-8", "-feature", "-unchecked", "-Xlog-reflective-calls") ++
     Seq("-Ystatistics")
  ```
  This will print compiler statistics.

2. Reload and check that the `-Ystatistics` option has been added

   ```console
   akka> reload
   akka> show akka-stream-typed/scalacOptions
   [info] * -Yrangepos
   [info] * -Xfatal-warnings
   [info] * -Ywarn-nullary-unit
   ...
   [info] * -Ystatistics
   ...
   ```

2. Export the build with `bloopInstall`

   ```console
   akka> bloopInstall
   ```
   The `-Ystatistics` option should now be added to the `.bloop/akka-stream-typed.json` file:

   ```json
   {
       "version" : "1.1.2",
       "project" : {
           "name" : "akka-stream-typed",
           ...
           "scala" : {
               ...
               "options" : [
                   ...
                   "-Ystatistics",
                   ...
               ],
               ...
           }
       }
   }
   ```

3. Compile the project and pipe the results to a `statistics.txt` file.

   ```console
   akka$ bloop clean
   akka$ bloop compile --pipeline akka-stream-typed &> statistics.txt
   ```

Examine the contents of `statistics.txt`.  You should be able to see each of the different compiler phases.

```
...
  xsbt-analyzer               : 1 spans, ()13.062ms (0.0%)
  silencerCheckUnused         : 1 spans, ()0.413ms (0.0%)
  jvm                         : 1 spans, ()1947.76ms (6.2%)
  delambdafy                  : 1 spans, ()137.977ms (0.4%)
  cleanup                     : 1 spans, ()94.163ms (0.3%)
  mixin                       : 1 spans, ()265.534ms (0.8%)
  flatten                     : 1 spans, ()74.037ms (0.2%)
  constructors                : 1 spans, ()229.355ms (0.7%)
  lambdalift                  : 1 spans, ()212.884ms (0.7%)
  posterasure                 : 1 spans, ()85.13ms (0.3%)
  erasure                     : 1 spans, ()1345.332ms (4.3%)
  explicitouter               : 1 spans, ()326.483ms (1.0%)
  specialize                  : 1 spans, ()399.752ms (1.3%)
  tailcalls                   : 1 spans, ()96.026ms (0.3%)
  fields                      : 1 spans, ()115.938ms (0.4%)
  uncurry                     : 1 spans, ()449.221ms (1.4%)
  refchecks                   : 1 spans, ()1201.219ms (3.8%)
  xsbt-dependency             : 1 spans, ()683.049ms (2.2%)
  xsbt-api                    : 1 spans, ()1142.043ms (3.6%)
  picklergen                  : 1 spans, ()2.296ms (0.0%)
  pickler                     : 1 spans, ()87.16ms (0.3%)
  extmethods                  : 1 spans, ()25.686ms (0.1%)
  superaccessors              : 1 spans, ()128.316ms (0.4%)
  patmat                      : 1 spans, ()725.54ms (2.3%)
  silencer                    : 1 spans, ()47.432ms (0.2%)
  typer                       : 1 spans, ()20033.307ms (64.0%)
  packageobjects              : 1 spans, ()14.386ms (0.0%)
  namer                       : 1 spans, ()106.585ms (0.3%)
  parser                      : 1 spans, ()1325.116ms (4.2%)
#total compile time           : 1 spans, ()31325.93ms
...
```

## The typer

Most projects with long compile times have problems with the typer.  The typer phase of `akka-streams-typed` is relatively small at 64%, and unlikely to have problems.
Let's examine it in more detail anyway.

We will use the [scalac-profiling](https://github.com/scalacenter/scalac-profiling) plugin.

1. In `akka/project/AkkaBuild.scala` add the compiler plugin to the `defaultSettings` just under the addition of the `DefaultScalacOptions`

   ```console
    scalacOptions in Compile ++= DefaultScalacOptions,
    addCompilerPlugin("ch.epfl.scala" %% "scalac-profiling" % "1.0.0"),
   ```

2. Find the root of your akka repository

  ```console
  akka$ pwd
  /home/zainab/lsug/akka
  ```

3. In `akka/project/AkkaBuild.scala` add the following options to the `DefaultScalacOptions`.  Replace `/home/zainab/lsug/akka` with the root of your akka repository.

   ```scala
   final val DefaultScalacOptions = Seq("-encoding", "UTF-8", "-feature", "-unchecked", "-Xlog-reflective-calls") ++
     Seq(
       "-Ystatistics",
       "-Ycache-plugin-class-loader:last-modified",
       "-P:scalac-profiling:no-profiledb",
       "-P:scalac-profiling:show-profiles",
       "-P:scalac-profiling:sourceroot:/home/zainab/lsug/akka"
     )
   ```

4.  In the sbt shell, clean and compile `akka-stream-typed`

    ```console
    akka> reload
    akka> project akka-stream-typed
    akka> clean
    akka> compile
    ```

    This should generate a `.flamegraph` file in `akka/akka-stream-typed/target/classes/META-INF/profiledb/graphs/`.

4. Navigate to `akka/akka-stream-typed/target/classes/META-INF/profiledb/graphs/` and find the name of the flamegraph

   ```console
   akka$ cd akka-stream-typed/target/classes/META-INF/profiledb/graphs
   graphs$ ls
   implicit-searches-1563721527947.flamegraph
   graph$ realpath implicit-searches-1563721527947.flamegraph
   /home/zainab/lsug/akka/akka-stream-typed/target/classes/META-INF/profiledb/graphs/implicit-searches-1563721527947.flamegraph
   ```

4.  We now need to process the `.flamegraph` file into a flamegraph.  Clone the `scalac-profiling` repository.

    ```console
    lsug$ git clone https://github.com/scalacenter/scalac-profiling.git --single-branch --depth=1
    lsug$ cd scalac-profiling
    scalac-profiling$
    scalac-profiling$ git submodule update --init
    ```

    Navigate into the `Flamegraph` directory

    ```console
    scalac-profiling$ cd Flamegraph
    ```

4. Generate a flamegraph using the `flamegraph.pl` script.

   Replace `/home/zainab/lsug/akka/akka-stream-typed/target/classes/META-INF/profiledb/graphs/implicit-searches-1563721527947.flamegraph` with the fill path to your flamegraph file.

    ```console
    Flamegraph$ ./flamegraph.pl \
    --hash --countname="μs" \
    --color=scala-compilation \
    /home/zainab/lsug/akka/akka-stream-typed/target/classes/META-INF/profiledb/graphs/implicit-searches-1563721527947.flamegraph \
     > akka-stream-typed.svg
    ```

5. Open the `akka-stream-typed.svg` file in your browser.

More information on this graph can be found in the [Scala blog post on scalac profiling](https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html).

Notice that there's a large red chunk labelled `Unit => akka.actor.typed.ActorRef`.  This means that many implicit searches for `ActorRef` result in failures.

## Xlog-implicits

To figure out why these implicit searches are failing, add the `-Xlog-implicits` compiler option.

1. In `akka/project/AkkaBuild.scala` add the compiler option to the `DefaultScalacOptions`.

   ```scala
   final val DefaultScalacOptions = Seq("-encoding", "UTF-8", "-feature", "-unchecked", "-Xlog-reflective-calls") ++
     Seq(
       "-Ystatistics",
       "-Xlog-implicits",
       "-Ycache-plugin-class-loader:last-modified",
       "-P:scalac-profiling:no-profiledb",
       "-P:scalac-profiling:show-profiles",
       "-P:scalac-profiling:sourceroot:/home/zainab/lsug/akka"
     )
   ```

2. Reload and check that the `-Xlog-implicits` option has been added

   ```console
   akka> reload
   akka> show akka-stream-typed/scalacOptions
   ...
   [info] * -Xlog-implicits
   ...
   ```

2. Export the build with `bloopInstall`

   ```console
   akka> bloopInstall
   ```
   The `-Xlog-implicits` option should now be added to the `.bloop/akka-stream-typed.json` file:

   ```json
   {
       "version" : "1.1.2",
       "project" : {
           "name" : "akka-stream-typed",
           ...
           "scala" : {
               ...
               "options" : [
                   ...
                   "-Xlog-implicits",
                   ...
               ],
               ...
           }
       }
   }
   ```

3. Compile the project and pipe the results to a `implicits.txt` file.

   ```console
   akka$ bloop clean
   akka$ bloop compile --pipeline akka-stream-typed &> implicits.txt
   ```

Open the `implicits.txt` file. You'll notice the following messages

```
 akka-actor/src/main/scala/akka/actor/dsl/Inbox.scala:126:62
      senderFromInbox is not a valid implicit value for akka.actor.ActorRef because:
      hasMatchingSymbol reported error: could not find implicit value for parameter inbox: Inbox.this.Inbox
      L126:             case Some(q) ⇒ { clientsByTimeout -= q; q.client ! msg }
```

This is a rather cryptic message.  Let's decompose it

```
 akka-actor/src/main/scala/akka/actor/dsl/Inbox.scala:126:62
...
      L126:             case Some(q) ⇒ { clientsByTimeout -= q; q.client ! msg }
```

This means that an implicit search was performed at line 126, character 62 of `Inbox.scala`.
This corresponds to the `!` method on the line

```scala
            case Some(q) ⇒ { clientsByTimeout -= q; q.client ! msg }
```

This is defined in the trait `ScalaActorRef` in `akka/akka-actor/src/main/scala/akka/actor/ActorRef.scala`

```scala
  def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit
```

`...is not a valid implicit value for akka.actor.ActorRef` implies that the compiler was searching for an implicit value of type `ActorRef`.

`...senderFromInbox is not a valid implicit value...` implies that the compiler tested the method `senderFromInbox`.  This is defined in `Inbox.scala`:

```scala
  implicit def senderFromInbox(implicit inbox: Inbox): ActorRef = inbox.receiver
```

`could not find implicit value for parameter inbox: Inbox.this.Inbox` implies that this test failed because there was no implicit `inbox` in scope.

This test could be avoided by defining `senderFromInbox` in a more appropriate location.

## Profiling implicit induction

The `akka-stream-typed` project doesn't use any implicit induction, so we won't go into details on profiling it here.  You can learn more about profiling implicit induction in the [Scala blog post on scalac profiling](https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html).

# Conclusion

 - Split up your project
 - Pipeline
 - Delete your unused code
 - Upgrade Scala
 - Use Graal
 - Profile your typer
