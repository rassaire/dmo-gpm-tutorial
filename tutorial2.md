
# Shape and pose vectors

We are interested in modelling transformations of shape and pose features.
We need to learn how to defined them and how minipulate point positions. We introcture a vector strcuture of hsape and pose features

The following package needed to define data structure are imported
```Scala
import multobjectmodels.ShapeAndPoseVector
import scalismo.geometry.{EuclideanVector, Point3D, _3D}
```

```Scala
val shapeVec1: EuclideanVector[_3D]= Point3D(4.0, 5.0, 6.0).toVector
val poseVec1: EuclideanVector[_3D]= Point3D(4.0, 5.0, 6.0).toVector

val shapeVec2: EuclideanVector[_3D]= Point3D(4.0, 5.0, 6.0).toVector
val poseVec2: EuclideanVector[_3D]= Point3D(4.0, 5.0, 6.0).toVector

val v1:ShapeAndPoseVector[_3D] = ShapeAndPoseVector(shapeVec1,poseVec1)
val v2:ShapeAndPoseVector[_3D] = ShapeAndPoseVector(shapeVec2,poseVec2)
```
The difference between two ShapeAndPoseVectors
```Scala
val v=ShapeAndPoseVector(shapeVec1-shapeVec2,poseVec1-poseVec2)
```
one feature class can be changed while the other is fixed 
```Scala
val v=ShapeAndPoseVector(shapeVec1-shapeVec2,poseVec1)
    //or
val v=ShapeAndPoseVector(shapeVec2,poseVec1+poseVec2)
```
