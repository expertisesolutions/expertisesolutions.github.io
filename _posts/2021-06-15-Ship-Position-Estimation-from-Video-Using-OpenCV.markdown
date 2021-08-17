---
layout: post
title:  "Ship Position Estimation from Video using OpenCV"
date:   2021-06-15 16:30:00 -0300
categories: opencv detection tracking computer-vision pose-estimation camera-estimation blender python
image: "/assets/img/2021-06-15-Ship-Position-Estimation-from-Video-Using-OpenCV.png"
share-img: "/assets/img/2021-06-15-Ship-Position-Estimation-from-Video-Using-OpenCV.png" 
author: João Antônio Cardoso
---

Recently we jump into a challenge to explore a specific application of computer vision: 
how can we estimate a top-view position of a ship from a given video from a fixed perspective camera?
After some work we found our way, reaching what seems to be a quite reasonable result. 

The following video is a simulation of our real-time processing running through a 60x speed video. 
On the left, we have the perspective view while on the right, the estimated top-view. The green rectangle
is the tracked ship, the blue dot is the estimated position along its coordinates (relative to the perspective camera) in red:

<video width="100%" controls autoplay loop muted allowfullscreen preload='metadata'>
    <source src="/assets/mp4/2021-06-15-Ship-Position-Estimation-from-Video-Using-OpenCV-Snippet.mp4" type="video/mp4">
    <p>Your browser does not support the video element.</p>
</video>

# The pipeline

To better explain whats going on here, we will breafily explain our pipeline for this solution:

![](/assets/img/2021-06-15-Pipeline.png)

First of all, we wouldn't be calling it a challenge if the footage wasn't nearly real footage. 
For sure the chosen footage needs to have great image quality, but its way more fun if there are no available 
information about the camera, the lenses, or its location, so all that needed to be estimated.
For this application, we are using a small section from a Live video of the YouTube channel [BroadwaveLiveCams](https://www.youtube.com/channel/UC6RbL0ZAyA_rc__Acbqh2mw).

But.. How can one estimate the camera pose with no prior information?
It's a bit tricky, but it is actually not that hard:

1. We first identified the site location and imported it to [Blender](https://www.blender.org/) using
an open-source plugin called [BlenderGIS](https://github.com/domlysz/BlenderGIS), that gives us a near-enough 3D reconstruction
for our location:
![](/assets/img/2021-06-15-Blender-GIS.png)

2. We took a walk around the site using _Google Street View_ looking for a building that could give us a similar frame, and 
luckily in this case we had only one candidate:
![](/assets/img/2021-06-15-StreetView.png)

3. Using the open source software called [fSpy](https://fspy.io/) we manually matched the camera pose.
![](/assets/img/2021-06-15-fSpy.png)

4. Then we imported to Blender (using [fSpy-Blender](https://github.com/stuffmatic/fSpy-Blender) add-on) and we confirmed our probable location:
![](/assets/img/2021-06-15-fSpy-Blender.png)

5. Further, we also explored another approach: getting the mached 2D and 3D positon of reference points (in red) on the reference image 
and used OpenCV's routine `cv.solvePnP()` to find the camera pose, then we reproject those 3D points to the camera, giving us the blue 
circles, that has its centers on the reprojected 2D point, and its radius represents its relative error:
![](/assets/img/2021-06-15-OpenCV-solvePnP.png)  
Then when we imported this estimation to Blender we got something coherent with our probable location, but this doesn't 
seem to be as good as the other one:
![](/assets/img/2021-06-15-Testing-Position-from-solvePnP.png)

6. The satellite estimation is a bit easier given that it is so far away that it is almost an orthogonal view, so the exact altitude can be
compensated with the lens size, means that we can find nearly infinite combinations that will nearly match our needs.
![](/assets/img/2021-06-15-Satellite-Estimation.png)

## Ok, we have our scene on Blender, but we want to be able to process it with OpenCV!
OpenCV and Blender use different 3D orientation schemes, then you need to transform from one to another. 
<!-- The details with code for this tricky part is well described in this [other post](#.) -->

Blender has a very good Python API, so we wrote a script to deal with that and export our data to a text file.

On our Python scripts inside Blender we:
1. automatically generate 1000 random cubes in the scene
2. export to a file the 3D coordinates of the centroid of each cube
3. project the centroid of each cube to both cameras
4. export the pixel coordinates of each cube's centroid for both cameras to a file

On our Python scripts outside Blender we:
1. load the matched points
2. using those 2D points we run `cv.findFundamentalMat()` to get a mask to filter out the inlier points
3. using the filtered points we run `cv.findHomography()` to get a homography matrix between the cameras
4. using the homography between the cameras we run `cv.perspectiveTransform()` to transform the non-filtered main camera to the satellite view
5. compute the mean error between the original points and the projected ones.

The result can be seen in the following image. In red, the original points, in cyan, the reprojected points. The mean error was `0.54 px`.

![](/assets/img/2021-06-15-Reprojection-Error.png)

Fine, now we have the camera poses and a system that can project points from one to another, but...
## How to estimate the ship coordinates?

With almost the same technique that we used to transform points from one perspective to another, we can find a homography matrix to project
a pixel of a camera to the 3D water plane! See below:

1. define four 3D points on the water plane
2. project those points to the main camera using `cv.projectPoints()`
3. using the projected points, find the homography matrix between the 3D points and the projected points
4. now, we can use the homography to project a given pixel of the main camera to the water plane!

# Conclusion

In this post, we saw how a specific problem in the computer vision field can be implemented, needing some analytical insights, 
and some manual steps along with the programming job of building an application.

As expected, real cases of computer vision are always challenging and sometimes require more than knowledge of the powerful OpenCV.
