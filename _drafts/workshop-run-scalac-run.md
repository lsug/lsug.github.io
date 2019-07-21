---
layout: post
title:  "Workshop: Run Scalac, run!"
date:   2019-07-17 19:04:06 +0100
author: Zainab Ali
categories: jekyll update
---

# Prerequisites

This workshop involves an unwieldy amount of tools.

Please come with the following tools installed, as we won't have time to set these up within the workshop.

- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Java 8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- [sbt 1.2.8](https://www.scala-sbt.org/release/docs/Setup.html)
- [Scala 2.12.8 and 2.13.0](https://www.scala-lang.org/download/)
- [Zipkin](https://zipkin.io/pages/quickstart.html) - either the docker image, jar or source code

If you have time, it would be far, far better to install these tools too
- [graphviz](https://www.graphviz.org/download/)
- [bloop](https://scalacenter.github.io/bloop/setup)
- [GraalVM](https://www.graalvm.org/downloads/) - either the Community or Enterprise edition
- [clone the Bloop GitHub repository](https://github.com/scalacenter/bloop)
- [clone the Scalac Profiling repository](https://github.com/scalacenter/scalac-profiling)
- [clone my fork of the Akka repository](https://github.com/zainab-ali/akka)

Start the bloop server beforehand, as this will download more dependencies.

# Checkout your project

For this workshop, we'll be using [akka](https://github.com/akka/akka).  The akka codebase is in pretty good condition, so we'll use an older version to make improvements on.

1. Clone **my fork** of akka

   ```console
   lsug$ git clone https://github.com/zainab-ali/akka.git
   lsug$ cd akka
   akka$
   ```

2. Checkout the branch `workshop-start`.

   ```console
   akka$ git checkout workshop-start
   ```

# Profiling

## Bloop

Bloop is a build server we can use to inspect the akka build.

1. Follow the [Bloop Installation Guide](https://scalacenter.github.io/bloop/setup) to install bloop.

2. Create the file `akka/project/bloop.sbt` with the contents

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

   ```console
   $ docker run -d -p 9411:9411 openzipkin/zipkin
   ```

2. Compile the `akka-stream-typed` project

   ```console
   akka$ bloop compile akka-stream-typed
   ```

   Bloop should detect Zipkin automatically and emit traces to it.

   You should be able to see Zipkin traces at [http://localhost:9411/zipkin/](http://localhost:9411/zipkin/)

Let's compile `akka-stream-typed` a few more times.

1. Create a file `warmup.sh` with the following contents

   ```sh
   #!/bin/bash

   for i in {1..10}; do
      echo "Iteration $i"
      bloop clean
      bloop compile $1
   done
   ```

2. Make this executable

   ```console
   akka $ chmod u+x warmup.sh
   ```

3. Run this with the project `akka-stream-typed`

   ```console
   akka $ ./warmup.sh akka-stream-typed
   ```

Take a look at the Zipkin traces.  Bloop uses the same JVM instance each time it compiles the project.  Notice that the compilation times get shorter and shorter as the JVM warms up.

### Examining a trace

Each akka project has a `scalac` span.  Some of these run sequentially, but some run in parallel.
The order in which the spans appear depends on the build graph.

Take a look at akka's build graph.

```console
$ bloop projects --dot-graph > build-graph.dot
$ dot -o build-graph.svg -Tsvg build-graph.dot
```

Open `build-graph.svg` using your browser.

## Build pipelining

Projects only depend on the `typer` phase of their dependency projects.  It's possible to start compilation of a project after its dependency projects have finished the typer phase, but before they've finished compiling completely.  This is termed *build pipelining*.  Bloop supports this with the `--pipeline` option.  Run `bloop compile` with the `--pipeline` option.

```console
$ for i in {1..10}; do
$   echo "Iteration $i"
$   bloop clean
$   bloop compile --pipeline akka-stream-typed
$ done
```

Sort the Zipkin traces by newest first.  You should notice that the `scalac` spans start before the previous ones have finished.

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

Even if you can't setup your environment, it's worth setting up the benchmarking suite.
Bloop contains a benchmarking suite based on JMH.

1. Checkout bloop

   ```console
   $ git clone https://github.com/scalacenter/bloop.git
   $ cd bloop
   $ git submodule update --init
   ```

2. Delete all lines in `bloop-community-build.buildpress`

3. Add an entry called `start` for the `workshop-start` akka branch:

   ```
   start,https://github.com/zainab-ali/akka.git#workshop-start
   ```

4. Enter `sbt` and run `exportCommunityBuild`

   ```console
   $ sbt
   sbt> exportCommunityBuild
   ```

## Running the benchmarks

6. In `sbt`, run
   ```console
   sbt> benchmarks/jmh:run .*HotBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=start -p projectName=akka-stream-typed
   ```

   This runs 7 warm up iterations and 5 main iterations.  Once completed, you should be given a score for each iteration

6. Bloop contains a `HotPipelinedBloopBenchmark` class for benchmarking pipelining
   ```console
   benchmarks/jmh:run .*HotPipelinedBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=start -p projectName=akka-stream-typed
   ```
   Do you notice a decrease in score?

# Upgrades

## Scala

Significant improvements to `scalac` are made in each new scala version. You can see the decrease in compile times in the [Grafana dashboard](https://scala-ci.typesafe.com/grafana/dashboard/db/scala-benchmark).

The akka `workshop-scala-2.13` branch contains a Scala 2.13 upgrade.

1. Add a community buildpress entry called `scala213` for the `workshop-scala-2.13` akka branch:

   ```
   scala213,https://github.com/zainab-ali/akka.git#workshop-scala-2.13
   ```

2. Export the community build once again

   ```console
   sbt> exportCommunityBuild
   ```

3. Run the benchmarks for the Scala 2.13
   ```console
   benchmarks/jmh:run .*HotPipelinedBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=scala213 -p projectName=akka-stream-typed
   ```

   Do you see a decrease in score?

## Graal VM

The scala compiler runs best on the Java 8 Graal VM.

1. Install Graal
2. Exit the bloop sbt shell

   ```console
   sbt> exit
   $
   ```
3. Switch your JVM to Graal
4. Enter the sbt shell and run the benchmarks once again
   ```console
   $ sbt
   sbt> benchmarks/jmh:run .*HotPipelinedBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=scala213 -p projectName=akka-stream-typed
   ```

   Do you see a decrease in score?

# Compile less code

scalac will be faster if it has less code to compile.  You can
 - use `scalafix` to delete unused imports and methods
 - keep an eye on the `scala-clean` project

# Profiling

We will now look into profiling `scalac` itself.

## -Ystatistics

1. In `project/AkkaBuild.scala`, add the `-Ystatistics` option to the `scalacOptions`.  This will print compiler statistics.
2. Export the build with `bloopInstall`
   ```console
   sbt> reload
   sbt> bloopInstall
   ```
   The `-Ystatistics` option should now be added to the `./bloop/akka-stream-typed.json` file.

3. Compile the project and pipe the results to a file.

   ```console
   $ for i in {1..10}; do
   $   echo "Iteration $i"
   $   bloop clean
   $   bloop compile --pipeline akka-stream-typed &> statistics-$i.log
   $ done
   ```

Examine the statistics in `statistics-10.log`.  You should be able to see each of the different phases and the time that they take to perform.

```
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
```

## The typer

Most projects that have long compile times have problems with the typer.  The typer phase of `akka-streams-typed` is fast, and unlikely to have problems.  Let's examine it in more detail anyway

1. Add the `scalac-profiling` plugin to `project/plugins.sbt`
   ```console
    addCompilerPlugin("ch.epfl.scala" %% "scalac-profiling" % "1.0.0")
   ```

2. Add the following options to the `scalacOptions` in `project/AkkaBuild.scala`.  Replace `REPOSITORY_ROOT` with the root of your akka repository.

   ```
     -Ycache-plugin-class-loader:last-modified
     -P:scalac-profiling:no-profiledb
     -P:scalac-profiling:show-profiles
     -P:scalac-profiling:sourceroot:$REPOSITORY_ROOT
   ```

3.  In the sbt shell, clean and compile `akka-stream-typed`

    ```console
    sbt> project akka-stream-typed
    sbt> clean
    sbt> compile
    ```

    This should generate a `.flamegraph` file in `akka/akka-stream-typed/target/classes/META-INF/profiledb/graphs/`.

4.  We now need to process the `.flamegraph` file into a flamegraph.

    ```console
    git clone https://github.com/scalacenter/scalac-profiling.git
    cd scalac-profiling
    git submodule update --init
    ```

    Navigate into the `Flamegraph` directory

    ```console
    cd Flamegraph
    ./flamegraph.pl \
    --hash --countname="μs" \
    --color=scala-compilation \
    $PATH_TO_FLAMEGRAPH_FILE \
     > akka-stream-typed.svg
    ```

5. Open the `akka-stream-typed.svg` file in your browser.

More information on this graph can be found in the [Scala blog post on scalac profiling](https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html).

Notice that there's a large red chunk labelled `Unit => akka.actor.typed.ActorRef`.  This means that many implicit searches for `ActorRef` result in failures.

## Xlog-implicits

To figure out why these implicit searches are failing, add the `-Xlog-implicits` compiler option.

Compile `akka-stream-typed` again.

```console
$ bloop clean
$ bloop compile --pipeline akka-stream-typed &> log-implicits.txt
```

You'll notice the following messages

```
 akka-actor/src/main/scala/akka/actor/dsl/Inbox.scala:126:62
      senderFromInbox is not a valid implicit value for akka.actor.ActorRef because:
      hasMatchingSymbol reported error: could not find implicit value for parameter inbox: Inbox.this.Inbox
      L126:             case Some(q) ⇒ { clientsByTimeout -= q; q.client ! msg }
```

This means that an implicit search failed in  `akka/actor/dsl/Inbox.scala` at line 126, column 62.
This corresponds to the line

```scala
            case Some(q) ⇒ { clientsByTimeout -= q; q.client ! msg }
```

In order to compile this code, we perform a search for a value of type `ActorRef`.  The log doesn't tell us which method requires the implicit `ActorRef`, but we can assume that it is `!`.
This is defined in the trait `ScalaActorRef` as

```scala
  def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit
```

When searching for the implicit `sender`, the compiler tests the function `senderFromInbox`.  This search fails, as there is no implicit `inbox` in scope.
This could be avoided by choosing a different location for the implicit `senderFromInbox` function.

The `akka-stream-typed` project doesn't use any implicit induction, so we won't go into details on profiling it here.  You can learn more about profiling implicit induction in the [Scala blog post on scalac profiling](https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html).

# Conclusion

 - Split up your project
 - Pipeline
 - Delete your unused code
 - Upgrade Scala
 - Use Graal
 - Debug your typer
