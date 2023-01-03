
To run the code in the tutorials, once you have set up [Using Scalismo](https://scalismo.org/docs/):
There are two ways to run the code. Either you configure it from a built.sbt file, or you clone it from github.
- First option: add this to your build.sbt file

```Scala
resolvers ++= Seq(
  Opts.resolver.sonatypeSnapshots,

  "shapmodelling-dmi" at https://shapemodelling.cs.unibas.ch/repo

)
 
libraryDependencies  ++= Seq(

            "ch.unibas.cs.gravis" % "scalismo-native-all" % "4.0.+",

            "ch.unibas.cs.gravis" %% "scalismo-ui" % "0.90.0",

            "io.github.cibotech" %% "evilplot" % "0.8.1",

            "latim" %% "dmfc-gpm" % "0.1"

)
```
Alternatively, downlaod this  project ([tutorial project](https://www.dropbox.com/s/f6d9cug2o23qyh6/dmfc-gpm-tutorial-project.zip?dl=0)). 


- Second option: Clone the project from github
- -
[Clone the project](https://github.com/rassaire/Dmfc-gpm)

You can open the project using  intelliJ IDE (here are some references : [Getting started with Scalismo in IntelliJ IDEA](https://scalismo.org/docs/ide)).
- run the code in the tutorials.
