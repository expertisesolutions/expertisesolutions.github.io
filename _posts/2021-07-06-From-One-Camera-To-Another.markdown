---
layout: post
title:  "From One Camera to Another"
date:   2021-07-06 10:00:00 -0300
categories: opencv detection tracking computer-vision python
image: "/assets/img/2021-07-06-From-One-Camera-To-Another-cover.png"
share-img: "/assets/img/2021-07-06-From-One-Camera-To-Another-cover.png"
author: João Antônio Cardoso
---

In this post, we will take a look at how to do a satellite or ‘top’ view (similar to bird-eyes) and with the same technique, how to estimate the position of a detected object. If you are new here, [take a look at our first post of this series, which is an overview of this particular case that we will be working on here](/2021-06-15-Ship-Position-Estimation-from-Video-Using-OpenCV).

To better contextualize, let’s summarize what we have so far, and link the posts where we see the relevant details of that specific part:
an accelerated video from a perspective camera filming the Miami Port, from YouTube channel [BroadwaveLiveCams](https://www.youtube.com/channel/UC6RbL0ZAyA_rc__Acbqh2mw);
an estimated pose and camera calibration, aka extrinsic and intrinsic parameters (see [Ship Position Estimation from Video Using OpenCV](/2021-06-15-Ship-Position-Estimation-from-Video-Using-OpenCV));
we got those camera parameters on Blender and OpenCV (see [From Blender to OpenCV](/2021-06-22-From-Blender-To-OpenCV));
an OpenCV algorithm detecting and tracking ships on the video (see [Simple and Fast Detection with Tracking](/2021-06-29-Simple-and-Fast-Detection-with-Tracking)).

From that, I would like to make it clear what we want here:
the ship’s position on the water plane as a coordinate;
the ship’s position as a point in both the perspective camera and the satellite image, which we call our top-view/satellite camera.

Although we are showing real code snippets here, this isn’t supposed to be a tutorial, so I should advise you to be prepared to grasp some details, like how the Blender project is structured, from code, or from zooming the images from other posts. But if you have any questions, we would be happy to answer them!

## From one camera to another

How can we project points from one camera to a second camera?

The truth is that it is not always easy or even possible, but fortunately, in our case, it is: as argued by [this answer on StackOverflow](https://stackoverflow.com/a/43565754/3850957), we can consider the following: if our points reside in a plane and if our cameras are calibrated (we know their intrinsic and extrinsic parameters), we can simply find a [Homography Matrix](https://docs.opencv.org/master/d9/dab/tutorial_homography.html#:~:text=Briefly%2C%20the%20planar%20homography%20relates%20the%20transformation%20between%20two%20planes%20(up%20to%20a%20scale%20factor)%3A)) between them, that can translate the points from one camera to another.

Before doing that, we need to do two things first:
Generate (on Blender) and export our reference points for both cameras
Load the exported data and the reference images

### Generating and exporting from Blender
In the past, we already covered [how to export cameras and objects from Blender to OpenCV](/2021-06-22-From-Blender-To-OpenCV), so here we will focus on generating the cubes and exporting the projected pixel coordinate to both cameras.

As we said, we need to generate objects in the Blender 3D world, project its centroid to the camera, and then export all this information to be used with OpenCV. Check below:

```python
# Get scene and scene collection references
scene = bpy.context.scene

# Recreate the collection
collectionName = "ReferencePoints"

# Generate random coordinates
rng = np.random.default_rng(12345)
npoints = 1000
x = rng.integers(low=-3200, high=2000, size=npoints)
y = rng.integers(low=-1000, high=2000, size=npoints)
z = np.ones(npoints) * 4.7
objectPoints = np.vstack((x, y, z)).T

# Create objects inside a collection
collection = createReferenceObjects(scene, collectionName, objectPoints)

# Get OpenCV location
nT = np.zeros(shape=(len(collection.objects), 3))
for i, object in enumerate(collection.objects):
  print(object.name, object.location)
  _, _, T = get_3x4_RT_matrix_from_blender(object)
  print("T\n", T)
  nT[i] = T

# Save the list of T into a plain text file
path = bpy.path.abspath("//")
file = f"{path}{collectionName}.txt"
np.savetxt(file, nT)
print(f"Saved to: \"{file}\".")

# Get pixel image location of each objectPoint for each Camera
mainCameraPixel = np.zeros(shape=(len(collection.objects), 2))
satCameraPixel = np.zeros(shape=(len(collection.objects), 2))
for i, object in enumerate(collection.objects):
  mainCameraPixel[i] = get_obj_centroid_from_camera(
    scene, 'Camera', object.location)
  satCameraPixel[i] = get_obj_centroid_from_camera(
    scene, 'Satellite Camera Perspective', object.location)

# Remove the created collection to reduce blender file size
removeObjects(scene, collectionName)

file = f"{path}{collectionName}MainCameraPixels.txt"
np.savetxt(file, mainCameraPixel, fmt="%d")
print(f"Saved to: \"{file}\".")

file = f"{path}{collectionName}SatCameraPixels.txt"
np.savetxt(file, satCameraPixel, fmt="%d")
print(f"Saved to: \"{file}\".")
```

The key functions that are not defined below you should get [in this previous post](/2021-06-22-From-Blender-To-OpenCV):

```python
def removeObjects(scene, collection_name):
  if collection_name in [collections.name for collections in bpy.data.collections]:
    collection = bpy.data.collections[collection_name]
    for obj in collection.objects:
      bpy.data.objects.remove(obj, do_unlink=True)
    bpy.data.collections.remove(collection)

def createReferenceObjects(scene, collection_name, objectPoints):
  """ Creates a collection called coolection_name inside scene,
  inside this collection, it creates a sphere for each point in objectPoints.
  It returns the created collection """

  removeObjects(scene, collection_name)
  collection = bpy.data.collections.new(collection_name)
  scene.collection.children.link(collection)

  # Create each object as a primitive sphere as a child of the collection
  for p, point in enumerate(objectPoints):
    # Create the object
    bpy.ops.mesh.primitive_uv_sphere_add(
      radius=1, 
      enter_editmode=False, 
      align='WORLD', 
      location=point, 
      scale=(1, 1, 1)
    )

    # The newly created object are automatically selected
    obj = bpy.context.selected_objects[0]

    # Rename and link to collection
    obj.name = f"ReferencePoint.{p}"
    collection.objects.link(obj)

    # Unlink from scene collection
    scene.collection.objects.unlink(obj)

  return collection

def get_obj_centroid_from_camera(scene, cam_name, coord):
  cam = bpy.data.objects[cam_name]
  scale = scene.render.resolution_percentage / 100
  point = world_to_camera_view(scene, cam, coord)
  point = np.array([
    point[0] * scale * scene.render.resolution_x,
    (1.0 -point[1]) * scale * scene.render.resolution_y
  ], dtype=int)
  print(point)

  return point
```


Now we should have the necessary data to work on the OpenCV side:
Both images
Global location of the boxes’ centroids
Projected centroids to both cameras
Intrinsic and Extrinsic camera parameters

### Loading the necessary data

```python
# Fix scaling problems when importing data from Blender
blenderSceneSize = (1920, 1080)
openCVSceneSize = (1280, 720)
scaleFactor = openCVSceneSize[0] / blenderSceneSize[0]

# Load Main Image
mainImg = cv.imread(“main_camera_view.jpg”)
mainImg = cv.resize(cv.cvtColor(mainImg, cv.COLOR_BGR2RGB), openCVSceneSize)

# Load Sattelite Image
satImg = cv.imread("satellite_view.jpg")
satImg = cv.resize(cv.cvtColor(satImg, cv.COLOR_BGR2RGB), openCVSceneSize)

# Load Main Camera Parameters, exported from Blender
mainProjectionMatrix = np.loadtxt('Main Camera.txt')
(mainIntrinsic, 
 mainRotationMatrix, 
 mainHomogeneousTranslationVector
) = cv.decomposeProjectionMatrix(mainProjectionMatrix)[0:3]
mainCameraPose = -cv.convertPointsFromHomogeneous(
  mainHomogeneousTranslationVector.T)
mainCamR = Rot.from_matrix(mainRotationMatrix)
mainCameraQuaternions = mainCamR.as_quat()
mainRvec, mainTvec = fromWorldPose(mainCameraPose, mainCameraQuaternions)

# Load Satellite Camera Parameters, exported from Blender
satProjectionMatrix = np.loadtxt('Satellite Camera.txt')
(satIntrinsic, 
 satRotationMatrix, 
 satHomogeneousTranslationVector
) = cv.decomposeProjectionMatrix(satProjectionMatrix)[0:3]
satCameraPose = -cv.convertPointsFromHomogeneous(
  satHomogeneousTranslationVector.T)
satCamR = Rot.from_matrix(satRotationMatrix)
satCameraQuaternions = satCamR.as_quat()
satRvec, satTvec = fromWorldPose(satCameraPose, satCameraQuaternions)

# Load 3D Reference Objects Points, exported from Blender
objectPoints = np.loadtxt('ReferencePoints.txt')

# Get Projected Points for the Main Camera, exported from Blender
mainPoints = np.loadtxt(
  'ReferencePointsMainCameraPixels.txt', 
  dtype=float).reshape(-1, 2) * scaleFactor

# Get Projected Points for the Satellite Camera, exported from Blender
satPoints = np.loadtxt(
  'ReferencePointsSatCameraPixels.txt', 
  dtype=float).reshape(-1, 2) * scaleFactor
```

Now we can easily find the homography matrix and transform points from one camera to the other using a perspective transformation:

### The Homography part

```python
# Get Fundamental Matrix
F, maskF = cv.findFundamentalMat(mainPoints, satPoints, cv.RANSAC)

# Filter only inlier points
mainPointsFiltered = mainPoints[maskF.ravel()==1]
satPointsFiltered = satPoints[maskF.ravel()==1]

# Find homography between cameras and apply the perspective 
# transformation to get the equivalent points
homographyMatrix = cv.findHomography(mainPointsFiltered, satPointsFiltered)[0]
# Finally, the points on the Satellite Camera:
projectedSatPoints = cv.perspectiveTransform(
  mainPoints.reshape(1, -1, 2), homographyMatrix).reshape(-1, 2)
```

# Finding real-world ship position

Now, using the same principles, we can find the homography matrix between an arbitrary plane: here, the water plane, and its coordinates will match their physical coordinate relative to the camera:

Describe the water plane with at least 4 points
Project those points to the main camera
Find homography between the water plane points and the projected ones
apply

```python
pts_water_plane = np.float32([
  [0, 0, 0],
  [0, 1, 0],
  [1, 0, 0],
  [1, 1, 0],
])
pts_on_camera, _ = cv2.projectPoints(
  Pts_water_plane, rvec, tvec, intrinsic, None
)
floorHomographyMatrix, _ = cv2.findHomography(pts_on_camera, pts_water_plane)
```

```python
# Suppose points_to_project is the points you want to know the 
# coordinates on the water plane:
coords_in = np.hstack(
  [points_to_project / scale, np.ones((points.shape[0], 1), dtype=float)]
).T
coord_out = (self.floorHomographyMatrix @ coords_in).T
coord_out[:, 0:2] /= coord_out[:, 2]
coord_out[:, 2] = 0
```

# Conclusion

In this post, we show two different ways to use the homography matrix that may not be the most trivial examples we found on tutorials, and this is one of the reasons for this series of posts - it is not easy to find examples applied to real-world problems.

And here we close what we think was interesting to detail about our case where we explored computer vision to estimate the position of a ship only using a still camera’s video. 

Again, if you want to know more about this, please contact us. We would be glad to hear from you. 

We hope you enjoyed it. Stay tuned for the next series!

<video width="100%" controls autoplay loop muted allowfullscreen preload='metadata'>
    <source src="../assets/mp4/2021-06-15-Ship-Position-Estimation-from-Video-Using-OpenCV-Snippet.mp4" type="video/mp4">
    <p>Your browser does not support the video element.</p>
</video>

