
 
# Building shape and pose models for multple object family
The objective of this tutorial is to learn how to build a shape and pose model for several families of objects from matching meshes. Correspondence means that objects of the same family have been registered using the same reference mesh and that each object is at its original spatial position (position from the acquisition).
#Preparation
As in the previous tutorials, we start by importing some commonly used objects and initializing the system.
```Scala
import java.io.File
import breeze.linalg.DenseVector
import multobjectmodels._
import scalismo.common._
import scalismo.geometry._
import scalismo.io.{LandmarkIO, MeshIO}
import scalismo.mesh.TriangleMesh
import scalismo.mesh.TriangleMesh.domainWarp3D
import scalismo.transformations.Translation
import scalismo.ui.api.ScalismoUI

scalismo.initialize()
implicit val rng = scalismo.utils.Random(42)

val ui = ScalismoUI()
```
## Loading and preprocessing a dataset:
Let's load (and visualise) a set of humerus and scapula meshes from which we wish to model variations in shape and pose. We are interested in the shoulder joint, but the framework works with any joint (object complex) with a finite number of objects:
```Scala
val dataGroup = ui.createGroup("datasets")

val humerusMeshFiles = new java.io.File("path/humeri/").listFiles
val scapulaMeshFiles = new java.io.File("path/humeri/").listFiles

val humerusRotationCenterFiles = new java.io.File("path/humeri/").listFiles
val sacpulaRotationCenterFiles = new java.io.File("path/humeri/").listFiles

val (meshes, rotCenters, meshViews, rotCenterViews) = for (i<- 0 to meshFiles.size-1) yield {
  val scapulaMesh = MeshIO.readMesh(scapulaMeshFile(i)).get
  val humerusMesh = MeshIO.readMesh(humerusMeshFile(i)).get
  
  val scapulaRotcent=LandmarkIO.readLandmarksJson[_3D](scapulaRotationCenterFiles(i)).get
  val humerusRotcent=LandmarkIO.readLandmarksJson[_3D](humerusRotationCenterFiles(i)).get
  
  val scapulameshView = ui.show(dsGroup, scapulamesh, "scapulaMesh")
  val humerusmeshView = ui.show(dsGroup, humerusmesh, "humerusMesh")
  
  val scapularotCentView = ui.show(dsGroup,scapulaRotcent, "scapulaRotcent")
  val humerusrotCentView = ui.show(dsGroup,humerusRotcent, "humerusRotcent")
  
  (List(scapulaMesh, humerusMesh), 
  List(scapulaRotCent, humerusRotCent), 
  List(scapulaMeshView,humerusMeshView),
  List(scapulaRotCentView,humerusRotCentView)
  ) // return a tuple of the mesh and roation centers with their associated view
}) .unzip // take the tuples apart, to get a sequence of meshes and rotation centers as well as one of Views

```

## Bringing data to a reference frame:
We can see that the humeri and scapulae are distributed in space, however, these relative positions are not the physiological relative positions. They are due to the acquisition protocol, for example you can have the humerus of the same patient in the same (physiological) pose in different positions in space. We need to solve this problem by aligning all the scapulae to a reference rotational scapula and then applying the same rigid transformations to the scapula. This allows all the scapulae to be aligned while maintaining the spatial relationship between each scapula and the humerus, which is the pose variation that we want to model.


```Scala
val alignedDataGroup = ui.createGroup("datasets at the reference frame")
val referenceScapula=meshes(0)._1(0).head
val referenceLandmarks = referenceScapula.pointSet.pointIds
      .map(pi => Landmark[_3D](pi.id.toString, reference.domain.pointSet.point(pi)))
      .toSeq
val (alignedMeshes, alignedRotCenters, meshViews, rotCenterViews) = for (i<- 0 to meshFiles.size-1) yield {
 
  val warpedLandmark = referenceScapula.pointSet.pointsWithId
      .map(pi => Landmark[_3D](pi._2.id.toString, scapulaMesh(pi._2)))
      .toSeq
  val besttransform=LandmarkRegistration.rigid3DLandmarkRegistration(referenceLandmarks, warpedLandmark, referenceLandmarks.haed.point)
  
  val scapulaMesh = meshes(i)._1(0).transform(besttransform)
  val humerusMesh = meshes(i)._1(1).transform(besttransform)
  
  val scapulaRotcent=Seq(Landmark[_3D]("A",besttransform(meshes(i)._2(0))))
  val humerusRotcent=Seq(Landmark[_3D]("A",besttransform(meshes(i)._2(1))))
  
  val scapulameshView = ui.show(alignedDataGroup, scapulamesh, "scapulaMesh")
  val humerusmeshView = ui.show(alignedDataGroup, humerusmesh, "humerusMesh")
  
  val scapularotCentView = ui.show(alignedDataGroup,scapulaRotcent, "scapulaRotcent")
  val humerusrotCentView = ui.show(alignedDataGroup,humerusRotcent, "humerusRotcent")
  
  (List(scapulaMesh, humerusMesh), 
  List(scapulaRotCent, humerusRotCent), 
  List(scapulaMeshView,humerusMeshView),
  List(scapulaRotCentView,humerusRotCentView)
  ) // return a tuple of the mesh and roation centers with their associated view
}) .unzip // take the tuples apart, to get a sequence of meshes and rotation centers as well as one of Views

```
 You can see that joint meshes (couple scapula and humerus) have changed positions while keeping their relative postion. 
 
 Exercise: find the rotation angles (Euler's angles) of some humeri (hemeri brought to the reference frame) with respect to the first humerus mesh in the dataset.
## Computing logarithmic functions from data
 Similar to [Single shape and pose models] (tutorial4.md), to study shape variations, we need to extract shape and pose variations. This is achiveved by selecting two of the meshes (scapula and humerus) as a references to create a [Multiple object structures](tutorial1.md), then use the reference to compute the [logarithimc mapping](tutorial3.md). The logaritmic map use to compute a seqquence of shared deformation fields (containing shape and pose fature for both objects) over which the Gaussian proces will be computed. This is simply done by computing a logarithmin defromation fields  for MultibodyObject[_3D, TriangleMesh]  structures created from the rest of the datasets.
 
 ```Scala
val referenceScapula=meshes(0)._1(0).head
val referenceHumerus=meshes(0)._1(1).head
val refRotCentScapula=rotCenters(0)._2(0).head.point
val refRotCentHumerus=rotCenters(0)._2(1).head.point

val referenceMultibodyObject:MultibodyObject[_3D, TriangleMesh] = MultibodyObject(
      List(referenceScapula,referenceHumerus),
      List(refRotCentScapula,refRotCentHumerus),
      List(Translation(EuclideanVector3D(0.0, 0.1, 0.0))(refRotCentScapula,Translation(EuclideanVector3D(0.0, 0.1, 0.0))(refRotCentHumerus)
      
    )

val expLog=MultiobjectPoseExpLogMapping(referenceMultibodyObject)

val defFields = for (i <- 0 to meshes - 1) yield {

      val targMultiBody=
      MultibodyObject(meshes(i)._1,
       meshes(i)._2,
       meshes(i)._2
          )

 val df = DiscreteField[_3D, MultibodyObject[_3D, TriangleMesh]#DomainT, EuclideanVector[_3D]](referenceMultibodyObject, referenceMultibodyObject.pointSet.pointsWithId.toIndexedSeq.map(pt => targMultiBody.pointSet.point(pt._2) - pt._1))

expLog.logMapping(df)
    }

 ```
   Note that the deformation fields can be be interpolated through nearest-neighbourhood interpolation to make sure that they are defined on all the points of the reference mesh. 
   
  ```Scala
 val continuousFields = defFields.map(f => f.interpolate(NearestNeighborInterpolator()))
 ```  
 
 ## Building of shape and pose models of multiple object families
  Now that these shape and pose variations are projected into tangent (vector) space at the reference MultibodyObject using logathimic mapping, learning the shape and pose variations for multple object family from these deformation fields is done using a PGA.
 

```Scala
val MultiBodyShapeAndPosePGA = MultibodyShapeAndPosePDM(df,expLog)
```
## Sampling of shape and pose model of multiple object families
 We can retrieve random samples of meshes and rotation centers from the model by calling sample on the Gaussian process:
 
```Scala
val sample=MultiBodyShapeAndPosePGA.sample()
val randomMeshScaoulaSample:TriangleMesh[_3D]=sample.objects(0)
val randomMeshHumerusSample:TriangleMesh[_3D]=sample.objects(1)
val randomRotCentScapula:Point[_3D]=sample.rotCenters(0)
val randomRotCentHumerus:Point[_3D]=sample.rotCenters(0)
ui.show(randomMeshScapulaSample, "randomMeshScapulaSample")
ui.show(randomMeshHumerusSample, "randomMeshHumerusSample")
ui.show(Seq(Landmark[_3D]("scapula",randomRotCentSample)), "randomRotCentSample")
ui.show(Seq(Landmark[_3D]("humerus",randomRotCentSample)), "randomRotCentHumerus")
```
## Marginalise model of shape and pose of multiple object famlies
 
 One would like to obtain the shape model only or the pose model only. 
 The marginalisation property of a Gaussian process allows to obtain the distribution for a specific feature class. [Single shape ad pose  models](tutorial4.md)  is calculated by specifying the erefence of the specifi object family, from shape models and shape cane be computed as discussied in [tutorial 4](tutorial.md).
 We can obtain this distribution, by calling the marginal method on the model. We compute the shape and pose model of the humerus.
 
 ```Scala
 val ShapeAndPose:ShapeAndPosePDM[Traingle]=MultiBodyShapeAndPosePGA.transitionToSingleObject(referenceHumerus)
 ```
## Posterior model of shape and pose of multiple object famlies

Similar ot the posterior model in [tutorial 4] (tutorial4.md), let us use Gaussian processes for regression tasks for models of shape and pose of multiple object falimies.The framework also allows regression to any of the features included in the domain. One of the practical applications of this regression is to constrain a model at the desired relative pose as well as the reconstruction of a partial shape. To calculate the regression, the Gaussian process model assumes that deformation is only observed up to a certain uncertainty, which can be modelled by a normal distribution. The observed data is specified in terms of points and their identifiers.
```Scala
val littleNoise=0.2 // variance of the Gaussian noise N(0,0.2)
val observedData = IndexedSeq((PointId, Point[_3D]))
val MultiBodyShapeAndPosePosterior: MultibodyShapeAndPosePDM[TriangleMesh] = MultibodyShapeAndPosePDM.posterior(observedData,littleNoise)
```
