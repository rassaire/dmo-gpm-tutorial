
To run the code in the tutorials, once you have set up [Using Scalismo](https://scalismo.org/docs/):
-add thsi to your build.sbt file:
resolvers ++= Seq(
``` Scala
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
Alternatively, downlaod this  project ( https://github.com/rassaire). 
You can open the project using  intelliJ IDE (here are some references : [Getting started with Scalismo in IntelliJ IDEA](https://scalismo.org/docs/ide)).
- run the code in the tutorials.
