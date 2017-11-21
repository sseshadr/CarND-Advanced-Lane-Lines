## **Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/undistorted.png "Undistorted"
[image2]: ./output_images/pipeundistorted.png "Road Transformed"
[image3]: ./output_images/pipethresholded.png "Binary Example"
[image4]: ./output_images/pipewarped.png "Warp Example"
[image5]: ./output_images/pipelinesfitted.png "Fit Visual"
[image6]: ./output_images/pipeareaprojected.png "Output"
[image7]: ./output_images/pipetextannotated.png "Output Video Snapshot"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in cells 2-7 of the IPython notebook located in "./adLaneFinding.ipynb".

I did this in 2 steps. First run the calibration pipeline on one sample calibration image (cells 2-5) and then run it on all calibration images (cell 6-7). The steps are as follows:
* Preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  
* Prepare `imgpoints`. This will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.
* Use `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  
* Apply distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

From the calibration results, we can now apply distortion correction on any image captured by that camera. Here is an example of a test image and the undistorted version obtained using `cv2.undistort()`

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color (in the S plane of the HLS color space) and gradient thresholds (sobel X) to generate a binary image (cell 27 with the function `threshold_image()`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp_image()` (cell 28).  This function takes as inputs an image (`img`).  I chose to hardcode the source and destination points in the following manner:

```python
apex = 460
lo = 203
lm = 585
rm = 720
ro = 1127
src = np.array([[(lm,apex), 
                 (lo,imshape[0]), 
                 (ro,imshape[0]), 
                 (rm,apex)]], dtype=np.float32)
dst = np.float32([[(img_size[0] / 4), 0],
                  [(img_size[0] / 4), img_size[1]],
                  [(img_size[0] * 3 / 4), img_size[1]],
                  [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points. Note that the initial value of the source points were inspired from the roi calculation step in Project 1: Finding Lane Lines:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 720, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I fitted the lane lines with a second order polynomial (cell 29-30, function `fit_lanes()`). The following steps were performed:
* Threshold the warped image from the previous step. 
* Take a histogram of the bottom half of the image to see the concentration of white pixels. Use this for initial X locations of the left and right lanes.
* Do a sliding window search. Startin with the initial positions from above, move a constant sized window (margin fixed) to identify the non zero pixels in that window. These are the points that will be used to fit the polynomial.
* Use `np.polyfit()` to fit a second order polynomial for the points collected in the previous step.
* Visualize fitted line on top of the original thresholded image by generating test points and looking at the output of the fit function as shown below.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Then I measured the radius of curvature of the lanes (cell 31, function `measure_curvature()`). The following steps were performed:

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Finally, I projected back the lane area (cell 31, function `project_back()`). The following steps were performed:
* Draw a polygon on the warped coordinate system with the points used for visualizing fit.
* Perform inverse perspective transform on the image to get the area in the sensor coordinate system (front camera view).
* Add this as a layer on the undistorted non-transformed image to visualize as shown below:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

The code for this is found in cells 33-38. The function `process_image()` in cell 34 has all the essential steps that are applied to each frame in the video as follows:

* Undistort frame
* Warp to bird's eye
* Threshold to get binary lanes
* Fit lanes to 2nd order polynomial
* Measure radius of curvature of lanes
* Project lane area back on the image
* Add annotations to show curvature measurement

Example screenshot below:

![alt text][image7]

Here's a [link to my video result](./test_videos_output/project_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
