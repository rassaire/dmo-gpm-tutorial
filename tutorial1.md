# Data Structures
The data structures we are talking about are the multibody object structure and the domain with pose parameters. 
These data structures can be defined from the basic mesh structures, which are triangular and tetrahedral meshes.
Or they can simply be unstructured points defined from point clouds. The zip file containing the lollipops (triangular meshes) used in the tutorials can be found in [Data](https://www.dropbox.com/s/rnct1inun112c3c/LollipopData.zip?dl=0).

The following package needed to define data structure are imported 
```Scala
import scalismo.io.{LandmarkIO, MeshIO}
import java.io.File
import ShapeAndPoseModels.DomainWithPoseParameters
import scalismo.geometry._3D
import ShapeAndPoseModels.MultiBodyObject
import scalismo.mesh.TriangleMesh
```
Meshes can be read from a file using the method readMesh from the MeshIO:
```Scala
    val TriangleMesh = MeshIO.readMesh(new File("LollipopData/first_objects/Synth1_30_0.stl")).get
    val TetrahedralMesh = MeshIO.readTetrahedralMesh(new File("LollipopData/tetrahedral_meshes/Synth3_28_0grid.vtk")).get
 ```
## Single object structure
Let us now define our first data structure called Domainwithposeparameters. This data structure consists of a discrete domain (triangle mesh, tetrahedral mesh or unstructured point), a rotation criterion and a neutral point. The rotation criterion is the point that is invariant under the translation of the object. The centre of rotation depends on the object, e.g. for the humerus it is at the head of the humerus and is calculated using the sphere fitting method.  For the scapula it is on the glenoid surface and is also calculated using the sphere fitting method. The neutral point is used to track the rotation center when aligning two objects with the invariant point bieng the center of the mass. It is therefore only used when computing the object, for an object it is the same as the centre of rotation. The rotation points are read from the landmark files.

```Scala
    val RotCent = LandmarkIO.readLandmarksJson[_3D](new File("LollipopData/rotation_centers_first_object/rotCenterObj1 0.json")).get
    val DomainWithPoseParam=DomainWithPoseParameters(TriangleMesh,
      RotCent.head.point,
      RotCent.head.point
    )
```
## Multiple object structure
Let us now define our second data structure called MultibodyObject, wich is the extension of Domainwithposeparameters to multple objects. The multy body data structures consist of a list of discrete domains, a list their corresponding rotation ceneters and a list of their nuetral points.


Let us load two triangle meshes and their centres of rotation.
```Scala
    val TriangleMesh1 = MeshIO.readMesh(new File("LollipopData/first_objects/Synth1_30_0.stl.stl")).get
    val TriangleMesh2 = MeshIO.readMesh(new File("/LollipopData/second_objects/Synth1_1_0.stl")).get

    val rotCent1 = LandmarkIO.readLandmarksJson[_3D](new File("LollipopData/rotation_centers_first_object/rotCenterObj1 0.json")).get
    val rotCent2 = LandmarkIO.readLandmarksJson[_3D](new File("LollipopData\rotation_centers_second_object/1_30_0.json")).get
 ```
Defining the MultibodyObject.
```Scala
    val multibodyObject:MultiBodyObject[_3D, TriangleMesh] = MultiBodyObject(
      List(TriangleMesh1,TriangleMesh2),
      List(rotCent1.head.point,rotCent2.head.point),
      List(rotCent1.head.point,rotCent2.head.point)

    )
```
