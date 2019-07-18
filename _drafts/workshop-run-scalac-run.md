---
layout: post
title:  "Workshop: Run Scalac, run!"
date:   2019-07-17 19:04:06 +0100
categories: jekyll update
---

# Prerequisites

- sbt 1.2.7
- docker
- graphviz

# Checkout your project

For this workshop, we'll be using [akka](https://github.com/akka/akka).  Akka is in a pretty good condition, so we'll use an older version to make improvements on.

1. Clone akka

```console
$ git clone https://github.com/akka/akka.git
```

2. Checkout the older revision `c383f44`.

```console
$ cd akka
$ git checkout c383f44 -b lsug
```

This is an build from December 2018

# Profiling

## Bloop

1. Follow the [Bloop Installation Guide](https://scalacenter.github.io/bloop/setup) to install bloop.

2. Add the bloop sbt plugin to `akka/project/bloop.sbt`

   ```scala
   addSbtPlugin("ch.epfl.scala" % "sbt-bloop" % "1.3.2")
   ```

2. Open an sbt shell in the `akka` directory

   ```console
   $ sbt
   ```

   Keep it running!

3. Export the build

   ```console
   sbt> bloopInstall
   ```

You should now be able to use `bloop` to compile your project.

Check that you can call bloop

```console
$ bloop --help
```

Try listing akka's projects:

```console
$ bloop projects
```

### Viewing build traces

Bloop outputs build traces.  These can be viewed with Zipkin.

1. Pull and start zipkin.

   ```console
   $ docker run -d -p 9411:9411 openzipkin/zipkin
   ```

2. Compile akka

   ```console
   bloop compile akka-test
   ```

   Bloop should detect zipkin automatically.

   You should be able to see Zipkin traces at [http://localhost:9411/zipkin/](http://localhost:9411/zipkin/)

Let's compile akka a few more times.

```console
$ for i in {1..10}; do
$   echo "Iteration $i"
$   bloop clean
$   bloop compile akka-test
$ done
```

Notice that the compilation times getting shorter and shorter as the JVM warms up.

### Examining a trace

Each akka project has a `scalac` span.  Some of these run sequentially, but some run in parallel.
The order in which the spans appear depends on the build graph.

Take a look at akka's build graph.

```console
$ bloop projects --dot-graph > build-graph.dot
$ dot -o build-graph.svg -Tsvg build-graph.dot
```

Open `build-graph.svg` using your browser.

## Reorganizing the build graph

Compile the `akka` project with `bloop compile akka` and examine the zipkin trace.
You'll notice that some test projects are getting compiled too.  Try and work out why this is by looking at the build graph.

The `akka-bench-jmh` project depends on the `akka.persistence.PersistenceSpec` within the `akka-persistence-test` project.  Does it need to?
Try and remove the dependency on `akka-persistence-test` by moving the relevant code elsewhere.

# Benchmarking

By removing the `akka-persistence-test` dependency from `akka`, we should have sped up our main build.  We can only really tell this by careful benchmarking.

Commit your changes to an `lsug` branch

1. Checkout bloop

   ```console
   $ git clone https://github.com/scalacenter/bloop.git
   $ cd bloop
   $ git submodule update --init
   ```

2. Delete all projects in `bloop-community-build.buildpress`

3. Add an entry for `akka`:

   ```
   akka,https://github.com/akka/akka.git#42fb18936f768de63aac512bf6126eef7c6b73f5
   ```
4. Add an entry for your changes.  For example:

   ```
   akka-reorg,https://github.com/zainab-ali/akka.git#09015ebbc4fc8e5c4bc9004d77bf1873d66a0c95
   ```
5. Enter `sbt` and run `exportCommunityBuild`

   ```console
   $ sbt
   sbt> exportCommunityBuild
   ```
6. In `sbt`, run
   ```console
   benchmarks/jmh:run .*HotBloopBenchmark.* -wi 7 -i 5 -f1 -t1 -p project=akka -p projectName=akka-test
   ```

# Upgrades

## Scala

Significant improvements are made in each scala version.

## Graal

# Compile less code

scalac will be faster if it has less code to compile, and less imports to search through.

### Remove unused imports

The easiest way to do this is to use scalafix.  Add the scalafix plugin to your `plugins.sbt`:
