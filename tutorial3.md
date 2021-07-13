
# Logarithmic and Exponential mappings
Shape and pose models are models based on principal geodesic analysis (PGA), which is a non-linear extension of principal component analysis. In this tutorial we show how the logarithmic and exponential mappings required for PGA are computation.

The following package needed to define data structure are imported
```Scala
import ShapeAndPoseModels.{DomainWithPoseParameters, ShapeAndPoseVector, SinglePoseExpLogMapping}
import scalismo.common.DiscreteField
import scalismo.geometry.{EuclideanVector, Landmark, _3D}
import scalismo.io.{LandmarkIO, MeshIO}
import scalismo.mesh.TriangleMesh
import scalismo.transformations.Transformation
```

## Logand Exp mappings for DomainWithPoseparamter
Let's start by calling the DomainWithPoseparamter from [tutorial 1](tutorial1) again, which we now call targetDomainWithPoseParam. 
```Scala
    val triangleMesh = MeshIO.readMesh(new File("path/trianglemesh.stl")).get
    val RotCent = LandmarkIO.readLandmarksJson[_3D](new File("path/rotcenter.json")).get

    val targetDomainWithPoseParam=DomainWithPoseParameters(triangleMesh,
      RotCent.head.point,
      RotCent.head.point
    )
```
Let's laod another DomainWithPOseparamter, which we call referenceDomainWithPoseParam
```Scala
    val referenceMesh:TriangleMesh[_3D] = MeshIO.readMesh(new File("path/referencemesh.stl")).get
    val referenceRotCent:Seq[Landmark[_3D]] = LandmarkIO.readLandmarksJson[_3D](new File("path/refrotcenter.json")).get

    val referenceDomainWithPoseParam: DomainWithPoseParameters[_3D, TriangleMesh]=DomainWithPoseParameters(referenceMesh,
      referenceRotCent.head.point,
      referenceRotCent.head.point
    )
```
Define the log and expo mapping
```Scala
    val logExp:SinglePoseExpLogMapping[TriangleMesh]=SinglePoseExpLogMapping(referenceDomainWithPoseParam)
```
 Let's now compute the [defromation field](https://scalismo.org/docs/tutorials/tutorial3) represenating targetDomainWithPoseParam with respect to referenceDomainWithPoseParam. 
 ```Scala
    val df=DiscreteField[_3D, ({ type T[_3D] = DomainWithPoseParameters[_3D, TriangleMesh] })#T, EuclideanVector[_3D]](referenceDomainWithPoseParam,
      referenceDomainWithPoseParam.pointSet.pointsWithId.toIndexedSeq.map(pt =>targetDomainWithPoseParam.pointSet.point(pt._2) - pt._1))
 ````
Let us now calculate the log function of targetDomainWithPoseParam, which is also a defromatization field. The log function factorises the object's features into shape and pose features, and then projects the pose features into a vector space and the shape into te shape space.
```Scala
    val logDF: DiscreteField[_3D, ({type T[D] = DomainWithPoseParameters[D, TriangleMesh]})#T, ShapeAndPoseVector[_3D]]=logExp.logMappingSingleDomain(df)
```
Computation of the exp function of targetDomainWithPoseParam, which is a rigid transformation. The exp function projects the pose defromation field component from the vector space into the manifold.
```Scala
   val expDF: Transformation[_3D] = logExp.expMappingSingleDomain(df)
```
