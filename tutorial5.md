
 
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
Let's load (and visualise) a set of first and second lollipop meshes from which we want to model variations in shape and pose. We are interested in the lollipop joint (a joint consisting of a first and second lollipop object), but the framework works with any (object complex) joint with a finite number of objects:
```Scala
val dataGroup = ui.createGroup("datasets")

  val firstObjectMeshFiles = new java.io.File("").listFiles
  val secondObjectMeshFiles = new java.io.File("").listFiles

  val firstObjectRotationCenterFiles = new java.io.File("").listFiles
  val secondObjectRotationCenterFiles = new java.io.File("").listFiles

  val dataFiles = for (i<- 0 to firstObjectMeshFiles.size-1) yield {
    val firstObjectMesh = MeshIO.readMesh(firstObjectMeshFiles(i)).get
    val secondObjectMesh = MeshIO.readMesh(secondObjectMeshFiles(i)).get

    val firstObjectRotCent=LandmarkIO.readLandmarksJson[_3D](firstObjectRotationCenterFiles(i)).get
    val secondObjectRotCent=LandmarkIO.readLandmarksJson[_3D](secondObjectRotationCenterFiles(i)).get

    val firstObjectMeshView = ui.show(dataGroup, firstObjectMesh, "firstObjectMesh"+i)
    val secondObjectMeshView = ui.show(dataGroup, secondObjectMesh, "secondObjectMesh"+i)

    val firstObjectRotCentView = ui.show(dataGroup,firstObjectRotCent, "firstObjectRotCent")
    val secondObjectRotCentView = ui.show(dataGroup,secondObjectRotCent, "secondObjectRotCent")

    (List(firstObjectMesh, secondObjectMesh),
      List(firstObjectRotCent.head.point, secondObjectRotCent.head.point),
      List(firstObjectMeshView,secondObjectMeshView),
      List(firstObjectRotCentView,secondObjectRotCentView)
    ) // return a tuple of the mesh and roation centers with their associated view
  }
```

<!-- ## Bringing data to a reference frame:
We can see that the first and second objects are distributed in space, however, these relative positions are not the physiological relative positions. They are due to the acquisition protocol, for example you can have the humerus of the same patient in the same (physiological) pose in different positions in space. We need to solve this problem by aligning all the scapulae to a reference rotational scapula and then applying the same rigid transformations to the scapula. This allows all the scapulae to be aligned while maintaining the spatial relationship between each scapula and the humerus, which is the pose variation that we want to model.


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
  -->
 Exercise: find the rotation angles (Euler's angles) of some humeri (hemeri brought to the reference frame) with respect to the first humerus mesh in the dataset.
## Computing logarithmic functions from data
 As with [Single shape and pose models] (tutorial4.md), to study shape variations we need to extract shape and pose variations. This is done by selecting two of the meshes (first and second objects) as references to create [Multiple Object Structures] (tutorial1.md), and then using the reference to calculate the [Logarithmic Mapping] (tutorial3.md). The logarithmic map is used to compute a sequence of shared deformation fields (containing the shape and posture of both objects) over which the Gaussian process will be computed. This is done simply by calculating a logarithm of the deformation fields for the MultibodyObject[_3D, TriangleMesh] structures created from the rest of the datasets.

Translated with www.DeepL.com/Translator (free version)
 
 ```Scala
   val firstObjectReference=dataFiles.head._1(0)
    val secondObjectReference=dataFiles.head._1(1)
    val firstObjectRotCentRef=dataFiles.head._2(0)
    val secondObjectRotCentRef=dataFiles.head._2(1)

    val referenceMultibodyObject: MultiBodyObject[_3D, TriangleMesh] =  MultiBodyObject(
      List(firstObjectReference,secondObjectReference),
      List(firstObjectRotCentRef,secondObjectRotCentRef),
      List(Translation(EuclideanVector3D(0.0, 0.1, 0.0)).apply(firstObjectRotCentRef),Translation(EuclideanVector3D(0.0, 0.1, 0.0)).apply(secondObjectRotCentRef))

      )

    val expLog=MultiObjectPoseExpLogMapping(referenceMultibodyObject)

    val defFields = for (i <- 0 to dataFiles.size - 1) yield {

      val targMultiBody=
        MultiBodyObject(dataFiles(i)._1,
          dataFiles(i)._2,
          dataFiles(i)._2
        )

      val df = DiscreteField[_3D, MultiBodyObject[_3D, TriangleMesh]#DomainT, EuclideanVector[_3D]](referenceMultibodyObject, referenceMultibodyObject.pointSet.pointsWithId.toIndexedSeq.map(pt => targMultiBody.pointSet.point(pt._2) - pt._1))

      expLog.logMapping(df)
    }

 ```
   Note that the deformation fields can be interpolated by nearest-neighbour interpolation to ensure that they are defined on all points of the reference mesh. 
   
  ```Scala
 val continuousFields = defFields.map(f => f.interpolate(NearestNeighborInterpolator()))
 ```  
 
 ## Building of shape and pose models of multiple object families
  Now that these shape and pose variations are projected into the tangent (vector) space of the reference multibody object using logathimic mapping, the learning of shape and pose variations for the multibody object family from these deformation fields is performed using a PGA.

```Scala
 val multiBodyShapeAndPosePGA = MultiBodyShapeAndPosePGA(defFields,expLog)
```
## Sampling of shape and pose model of multiple object families
 We can recover random samples of meshes and centres of rotation from the model by calling the sample on the Gaussian process:
 
```Scala
    val sample=multiBodyShapeAndPosePGA.sample()
    val randomFirstObjectSample:TriangleMesh[_3D]=sample.objects(0)
    val randomSecondObjectSample:TriangleMesh[_3D]=sample.objects(1)
    val randomFirstObjectRotCent:Point[_3D]=sample.rotationCenters(0)
    val randomSecondObjectRotCent:Point[_3D]=sample.rotationCenters(1)

    ui.show(randomFirstObjectSample, "randomFirstObjectSample")
    ui.show(randomSecondObjectSample, "randomSecondObjectSample")
    ui.show(Seq(Landmark[_3D]("scapula",randomFirstObjectRotCent)), "randomFirstObjectRotCent")
    ui.show(Seq(Landmark[_3D]("humerus",randomSecondObjectRotCent)), "randomSecondObjectRotCent")
```
## Marginalise model of shape and pose of multiple object famlies
 
One wishes to obtain the shape model only or the pose model only. 
 The marginalisation property of a Gaussian process is used to obtain the distribution for a specific feature class. The [shape and pose model](tutorial4.md) is computed by specifying the origin of the specific object family, from the shape and pose models can be computed as discussed in [tutorial 4](tutorial.md).
 We can obtain this distribution, by calling the marginal method on the model. We calculate the shape and pose model of the second object.
 
 ```Scala
    val referenceDomainWithPoseParam=DomainWithPoseParameters(secondObjectReference,
      secondObjectRotCentRef,
      Translation(EuclideanVector3D(0.0, 0.1, 0.0)).apply(secondObjectRotCentRef)
    )

    val secondObjectTransitionModel:ShapeAndPosePGA[TriangleMesh]=multiBodyShapeAndPosePGA.transitionToSingleObject(referenceDomainWithPoseParam)

    val singleObjectsample=secondObjectTransitionModel.sample()
    val secondObjectSample:TriangleMesh[_3D]=singleObjectsample.domain
    ui.show(secondObjectSample, "secondObjectSample transition model")
 ```
## Posterior model of shape and pose of multiple object families

Similar ot the posterior model in [tutorial 4] (tutorial4.md), let us use Gaussian processes for regression tasks for models of shape and pose of multiple object falimies.The framework also allows regression to any of the features included in the domain. One of the practical applications of this regression is to constrain a model at the desired relative pose as well as the reconstruction of a partial shape. To calculate the regression, the Gaussian process model assumes that deformation is only observed up to a certain uncertainty, which can be modelled by a normal distribution. The observed data is specified in terms of points and their identifiers.
```Scala
val littleNoise=0.2 // variance of the Gaussian noise N(0,0.2)
val observedData = IndexedSeq((PointId, Point[_3D]))
val MultiBodyShapeAndPosePosterior: MultibodyShapeAndPosePDM[TriangleMesh] = MultibodyShapeAndPosePDM.posterior(observedData,littleNoise)
```
