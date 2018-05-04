## Project Writeup


---

**Advanced Lane Finding Project**

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
[chessboard_distorted]: ./distorted_chessboard.jpg "Distorted chessboard"
[chessboard_undistorted]: ./undistorted_chessboard.png "Undistorted chessboard"
[distorted]: ./distorted.png "Distorted scene"
[undistorted]: ./undistorted.png "Undistorted scene"

[binary]: ./binary.png "Binary"
[color_binary]: ./color_binary.png "Color binary"
[binary_warped]: ./binary_warped.png "Warp Example"
[masked_binary_region]: ./masked_binary_region.png "Masked binary"
[binary_region_with_mask]: ./binary_region_with_mask.png "Binary with mask"
[with_lines]: ./with_lines.png "lines"
[rect_w_lines]: ./rect_w_lines.png "rect and lines"
[rect_and_lines]: ./rect_and_lines.png "rect and lines"
[final]: ./final.png "Final"
[video1]: ./project_video_result.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration


The code for this step is contained in the calibrate() function at line 258 through 292 in the Pipeline class declared in  "./advanced_lane_finding.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to this test image

![alt text][chessboard_distorted]

 using the `cv2.undistort()` function (wrapped in the `Pipeline.undistort` method and obtained this result: 

![alt text][chessboard_undistorted]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][distorted]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines 28 through 61 in the main `Pipeline` class). I chose to convert the frame to HLS color space because both L and S channels proved to detect lines in a variety of lighting conditions and line colors (yellow or white). A x-gradient Sobel threshold is used, and combined with the H and L thresholds achieve a binary output as follows:

![alt text][color_binary]

Each color in the image represent the contribute of S-channel (red), L-channel (green), and x-threshold (blue). The image data actually used by the algorithm just uses one channel that comprises all three contributions like this:

![alt text][binary]

Subsequently, I chose to apply a region of interest mask with the following vertices `((0, 720),(550,450),(800,450),(1280,720))`, to isolate the portion of frame that should always contain pixels belonging to lane lines:

![alt text][binary_region_with_mask]

![alt text][masked_binary_region]

This should help in avoiding that the line finding algorithm will not be "distracted" by non-line elements.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `perspective_transform()`, which appears in lines 197 through 265 in the `Pipeline` class. The `perspective_transform()` function takes as inputs an image (`img`), and uses a transforrm matrix `M` calculated during the calibration step (this is because the calibration step is only executed once, so I thought it would not be necessary to perform the transformation more than once).  I chose to hardcode the source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][binary_warped]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To identify lines out of these lane-like pixels, a windowing approach was used. Starting from the bottom, a rectangular window is used to detect if there are many pixels (`50`) in the window, and if so, average their x coordinate, and that should yield a likely position of the line. This step is repeated by moving the window up, re-centering the rectangle around the newly found line (if any).
After this sequence of pixel is found, I fit a 2nd order polynomial through them, which approximate each line to a parabola. This part of the algorithm results in the following image, where in green are represented the windowed rectangles and in yellow the fitted polynomial. This algorithm is in lines `105` to `187` in the Pipeline class.

![alt text][rect_and_lines]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Using a second order polynomial makes easier to calculate the curvature of a line using well-known equations. See lines `208` to `218` for the steps.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines  `278` through `298` in the Pipeline class in the `perspective_transform` function.  Here is an example of my result on a test image:

![alt text][final]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_result.mp4)
To reduce noise and to make the line detection more robust to sporadic pixel-changes on the image, the lane lines information are updated at each frame, keeping 80% of old information.
This can be seen at lines `164` and `178` in the `Pipeline()` class. Essentially is implemented as
```python
self.l_line.best_fit = self.l_line.best_fit*0.8 + self.l_line.current_fit*0.2
```

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The approach used in this project was very simplistic and only works in very good conditions of the video. More challenging videos reveal that the algorithm is not robust enough to accurately detect lines under various situations.
There are multiple points where the algorithm could be improved.
The thresholding algorithm just uses one RGB to HLS conversion and takes two color channels and one gradient, this could be improved by using data from multiple color spaces and gradients. Also, if the asphalt sharply changes in color, as it is the case when driving on roads with old and new asphalt, it is very likely that these transitions may be detected as lines.
Sanity check the detected lines, and making sure that there are only two lines (left-right) with no abrupt changes in curvature/fit/width/distance may robustify the algorithm. 
