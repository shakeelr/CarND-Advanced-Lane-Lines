## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./writeup/undistort_checkerboard.png "Undistorted Checkerboard"
[image2]: ./writeup/undistort_example.png "Undistorted Example"
[image3]: ./writeup/perspective_transform.png "Perspective Transformed"
[image4]: ./writeup/thresholding.png "Thresholded"
[image5]: ./writeup/fitted.png "Fitted Visual"
[image6]: ./writeup/overlay.png "Lane Overlay"
[image7]: ./writeup/pipeline.png "Complete Pipeline"
[video1]: ./output_videos/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 3rd code cell of the IPython notebook located in "./project.ipynb" (or in lines # through # of the file called `some_file.py`).  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to one of the chessboard images using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The first stage of my pipeline (found in code cell 4) is distortion correction.  In this stage, I use the camera calibration coefficients obtained in the camera calibration step applied above to undistort images using `cv2.undistort()`.  An example of an undistorted image is shown below:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The second stage of my pipeline (found in code cell 5) was to do a perspective transform on the road to give me an overhead perspective of the road in front of the car.  To do a perspective transform I manually selected points corresponding with corners of the road in an undistorted image of a straight section of road and mapped them to a rectangle, as seen in the code snippet and table below:

```python
src_bottom_left = [200,720]
src_bottom_right = [1110, 720]
src_top_left = [582, 460]
src_top_right = [700, 460]
src = np.float32([src_bottom_left,src_bottom_right,src_top_left,src_top_right])

dst_bottom_left = [200,720]
dst_bottom_right = [1110, 720]
dst_top_left = [200, 0]
dst_top_right = [1110, 0]
dst = np.float32([dst_bottom_left,dst_bottom_right,dst_top_left,dst_top_right])
```

This resulted in the following source and destination points in the table below which I used to obtain the perspective transformation matrix, M, using the `cv2.getPerspectiveTransform(src, dst)` function.  I also switched the arguments and used `cv2.getPerspectiveTransform(dst, src)` to get the inverse perspective transform matrix:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 200, 720      | 200, 720      | 
| 1110, 720     | 320, 720      |
| 582, 460      | 200, 0        |
| 700, 460      | 1110, 0       |

With the M matrix, I'm then able to use the function `cv2.warpPerspective()` to warp the image to the new perspective.  I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image3]

#### 3.  Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The third step of my pipeline (found in code cell 7) was to apply binary thresholds to the perspective transformed road to highlight pixels corresponding with lane lines.  I experiemented with a number of different color and gradient thresholding methods but ultimately found I could obtain clear lane lines using an RGB threshold to identify white lines.  The RGB threshold values to identify white lines were r > 190, g > 190, and b > 190.  Yellow lines were more difficult to pick up, and after some experimenting in various color spaces I found yellow could easily be identified in the LAB color space using the b channel.  The b threshold value to pick up yellow lines in the LAB colorspace was b > 185.  The b channel of the LAB colorspace was normalized prior to applying the threshold.  Examples of thresholded lane lines are shown below:

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The 4th step of my pipeline (found in code cell 10) is to fit identified lane line pixels to a polynomial.  This is done using a sliding histogram window method.  Line line pixels in each window with a minimum number of pixels are appended to a list and a polynomial is fitted through it using the `np.polyfit()` method.  An image showing a visualization of the windows, identified pixels, and polynomial fitted through then is shown below:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The next step of my pipeline (found in code cell 11) is to calculate the radius of curvature and offset of the lane lines.  I calculate the radias of curvature using the equation:

`(1 + (2*a*y*ym_per_pix + v)**2)**1.5) / abs(2*c)`

where a, b, c are the coeffients of the polynomial derived in pipeline stage 4, y = 720 for the top of the image, and ym_per_pix is the number of meters per pixel in the y axis (assumed to be 30m/pixel).  The above equation is applied to both the left and right lane lines and averaged together to estimate the radias of curvature.

Offset is caculated by finding the x value for both lanes at the bottom of the image, finding the midpoint of the two lanes at that point, and subtracting that from the midpoint of the image.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The final step of my pipeline (found in code cell 12)is projecting the lanes back onto the road.  To do this I create blank image the same size as the warped image and fill the area between the fitted lane lines onto it.  I color that area green and unwarp it using the inverse perspective transform, which is to use `cv2.warpPerspective()` with the inverse matrix Minv as an argument, then overlay it onto the road using "cv2.addWeighted()" to get a transperant effect.  I also add text to the final image for the radias of curvature and offset.  The final result can be seen below:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

I created a class for tracking lines, `Line()` in code cell 13, which is used in the final image processing pipeline for smoothing.  The final pipeline is found in code cell 14.  Smoothing is achieved by storing the 5 most recent lane lines in the line class and averaging them together before projecting the averaged lanes back on the road.

Here's a [link to my video result](./output_videos/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

While the appraoch I took is fairly straightforward and seems to track the lanes reasonably well, has a few shortcomings that could be improved upon. First, values for radias of curvature, while they seem to be in a reasonable order of magnatude (2-3km radias of curvature on a section or road with a 1km radias of curvature), tend to be noisy - even with smoothing.  There are two main reasons for this:

1. The right lane is dashed, and the line that fits through it does not always fit well.  This causes the left and right lanes to deviate in curvature, throwing the average off.
2. In cases where there are dark shadows on the road, the lane pixels may be difficult to detect, which can also cause problems with polynomial fitting.

Both these issues could probably be mitigated using sanity checks.  For example making sure detecting lanes are parallel and have similar radii of curvature.  Sudden changes in lane curvature could also be a sign that the lanes were not being fitted correctly.
