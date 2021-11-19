
# Building Shape and Pose models for a Single object family
The aim of this tutorial is to learn how to build a static shape and pose model for a single object from matching meshes. In correspondence, here means objects have been registered using the same reference mesh and each object is at its original spatial position (position from the acquisition) 
#Preparation
As in the previous tutorials, we start by importing some commonly used objects and initializing the system.
```Scala
  import multobjectmodels.{DomainWithPoseParameters, ShapeAndPosePDM, SinglePoseExpLogMapping}
  import scalismo.common.{DiscreteField, PointId}
  import scalismo.common.interpolation.NearestNeighborInterpolator
  import scalismo.geometry._
  import scalismo.io.{LandmarkIO, MeshIO}
  import scalismo.mesh.TriangleMesh
  import scalismo.statisticalmodel.PointDistributionModel
  import scalismo.transformations.Translation
  import scalismo.ui.api.ScalismoUI

  scalismo.initialize()
  implicit val rng = scalismo.utils.Random(42)
  val ui = ScalismoUI()
```
## Loading and preprocessing a dataset:
Let us load ([download here](https://www.dropbox.com/s/t592zkjg9tu06xz/LollipopData.zip?dl=0)) and visualize a set of objects meshes based on which we would like to model shape and pose variation:
```Scala
 val dataGroup = ui.createGroup("datasets")

   val meshFiles = new java.io.File("LollipopData/second_objects/").listFiles
    val rotationCenterFiles = new java.io.File("LollipopData/rotation_centers_second_object/").listFiles


  val dataFiles = for (i<- 0 to meshFiles.size-1) yield {
      val mesh = MeshIO.readMesh(meshFiles(i)).get
      val rotcent=LandmarkIO.readLandmarksJson[_3D](rotationCenterFiles(i)).get
      val meshView = ui.show(dataGroup, mesh, "mesh"+i)
      val rotCentView = ui.show(dataGroup, mesh, "rotCent"+i)
      (mesh, rotcent, meshView, rotCentView) // return a tuple of the mesh and rotation centers with their associated view
    }
```
 You can see that the meshes are at different positions in space and they are in correspondence. This means that for each point on one of the second lollipop objects meshes, we can identify the corresponding point on the other meshes. The corresponding points are identified by the same point identifier. 
 
 Exercise: find the rotation angles of some meshes with respect to the fist meshes in the dataset.
## Building logarithmic functions from data
 In order to study shape variations, we need to factor the shape variations and the pose variations (rotation and translation). This is done by selecting one of the meshes as a reference to create a [DomainWithPoseParamter](tutorial1.md), and then using the reference to compute the [logarithmic mapping](tutorial3.md). The logarithmic map is used to compute a sequence of deformation fields on which the Gaussian process will be computed. This is done simply by computing a logarithm of the deformation fields for the [DomainWithPoseParamter](tutorial1.md) created from the rest of the datasets.
 
 ```Scala
  val reference = dataFiles.head._1
    val refRotCent=dataFiles.head._2


    val referenceDomainWithPoseParam=DomainWithPoseParameters(reference,
      refRotCent.head.point,
      Translation(EuclideanVector3D(0.0, 0.1, 0.0)).apply(refRotCent.head.point)
    )

    val singleExpLog=SinglePoseExpLogMapping(referenceDomainWithPoseParam)

    val defFields = for (i <- 0 to dataFiles.size - 1) yield {

      val targDomainWithPoseParam=DomainWithPoseParameters(dataFiles(i)._1,
        dataFiles(i)._2.head.point,
        dataFiles(i)._2.head.point
      )

      val df=DiscreteField[_3D, ({ type T[D] = DomainWithPoseParameters[D, TriangleMesh] })#T, EuclideanVector[_3D]](referenceDomainWithPoseParam,
        referenceDomainWithPoseParam.pointSet.pointsWithId.toIndexedSeq.map(pt =>targDomainWithPoseParam.pointSet.point(pt._2) - pt._1))

      singleExpLog.logMappingSingleDomain(df)

    }
 ```
   Note that the deformation fields can be be interpolated through nearest-neighbourhood interpolation to make sure that they are defined on all the points of the reference mesh. 
  ```Scala
 val continuousFields = defFields.map(f => f.interpolate(NearestNeighborInterpolator()))
 ```  
 
 ## Single shape and pose model building
  Now that these shape and pose variations are projected into tangent (vector) space at the reference using logathimic mapping, learning the shape and pose variations from these deformation fields is done using a PCA. This is essentially the PGA, which is the PCA in tangent space. The model is compouted using the ```ShapeAndPosePDM``` class.
 

```Scala
val shapeAndPoseModel: ShapeAndPosePDM[TriangleMesh] = ShapeAndPosePDM(defFields,singleExpLog)
```
## Single shape and pose model sampling

 We can retrieve random samples of meshes and rotation centers from the model by calling sample on the Gaussian process:
 
```Scala
val sample=shapeAndPoseModel.sample()
val randomMeshSample:TriangleMesh[_3D]=sample.domain
val randomRotCent:Point[_3D]=sample.rotCenter
ui.show(randomMeshSample, "randomMeshSample")
ui.show(Seq(Landmark[_3D]("",randomRotCent)), "randomRotCent")
```
## Single shape and pose model marginalisation
 
 One would like to obtain the shape model only or the pose model only. 
 The marginalisation property of a Gaussian process allows to obtain the distribution for a specific feature class. The shape model is calculated as a point distribution model, while the pose model is again a ShapeAndPose  model, but with the shape set to the mean shape.
 We can obtain this distribution, by calling the marginal method on the model:
 ```Scala
 val MarginalshapeModel: PointDistributionModel[_3D, TriangleMesh]=shapeAndPoseModel.shapePDM
 val MarginalPoseFromShapeAndPose: ShapeAndPosePDM[TriangleMesh]=shapeAndPoseModel.PosePGA
 ```
## Single shape ad pose  model posterior

Let us now use Gaussian processes for regression tasks and experiment with the concept of posterior shape and pose models.  This shows how these tools can be applied to construct a partial shape reconstruction as well as pose estimation from shape observation. Since the ShapeAndPosePDM is simply a wrapper around a GP, similar to [Posterior shape models](https://scalismo.org/docs/tutorials/tutorial8), the same posterior functionality is available for shape and pose models. To calculate the regression, the Gaussian process model assumes that deformation is only observed up to a certain uncertainty, which can be modelled by a normal distribution. The observed data is specified in terms of points and their identifiers.
```Scala
  val littleNoise=0.2 // variance of the Gaussian noise N(0,0.2)
  val observedData:IndexedSeq[(PointId, Point[_3D])]= ???
  val ShapeAndPosePosterior: ShapeAndPosePDM[TriangleMesh] = shapeAndPoseModel.posterior(observedData,littleNoise)
```
