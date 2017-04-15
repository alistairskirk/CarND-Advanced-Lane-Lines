**Advanced Lane Finding Project**
Udacity Self Driving Car NanoDegree Project
Alistair S. Kirk April 10 2017

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

[csquares]: output_images/1_calibrationsquares.png "Calibration Squares"
[undsquares]: output_images/2_undistortedsquares.png "Undistorted Squares"
[perspsquares]: output_images/3_undistortedperspectivesquares.png "Perspective Squares"
[roadpersp]: output_images/4_roadperspectivewarp.png "Road Perspective Example"
[sobel]: output_images/5_sobelHSLthresh.png "Sobel & HSL Threshold"
[binarywarp]: output_images/6_binarywarpedpipeline.png "Binary Warp Pipeline"
[window]: output_images/7_windowedhistogram.png "Windowed Lane Find"
[continue]: output_images/8_continuedlanefind.png "Continued Lane Find"
[hud]: output_images/9_huddisplay.png "HUD"
[test1]: output_images/10_testimage1.png "Test Image 1"
[test2]: output_images/11_testimage2.png "Test Image 2"
[test3]: output_images/12_testimage3.png "Test Image 3"

[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "P4 - ASK.ipynb".  

Following from the tutorial given in the course; I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

An example of the square finding algorithm is shown here:

![csquares][csquares]

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function as shown in the second code block in the notebook.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![undsquares][undsquares]

This doesn't correct for perspective, so I made a unwarp function as seen in the third code block in the notebook, with an example shown below:
![perspsquares][perspsquares]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
Distortion correction for the road required some assumptions to be made about the trapezoidal masking frame. The road unwarping function in the fourth code block in the notebook, requires vertices to be supplied from the road mask function. This uses creates a trapezoid based on the dimensions of the input image and warps the road image to provide a simulated top-down view of the road and results in vertical lane lines for a straight parallel lane. The parameters of the road mask turn out to be a critical assumption that affects performance in harder challenge videos, and a future work should consider estimation of the road horizon to deal with tight turns and sharp hills.
I apply the distortion correction to one of the test road images:
![roadpersp][roadpersp]

####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of color and gradient thresholds to generate a binary image as shown in the fifth and sixth code blocks of the notebook. I selected the S Channel from the HSV conversion of the input image to detect most of the desired yellow and white lane lines. I then pass the S Channel through a Sobel X and Y Gradient filter, and use magnitude and direction threshold functions to further refine the lane lines. I also added a third layer to check for white pixels (RGB > 200) because some of the greyish roadlines in the challenge videos were not being picked up by just the S Channel. Lighting conditions of the road and environment have a strong effect on the detection of the lane lines, which means future work will require further refinement of these thresholding algorithms.

Here's an example of my output for this step:

![sobel][sobel]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `binary_warp()`, which appears in the seventh code cell of the IPython notebook.  The function takes as inputs an image (`img`), generates a roadmask using the previous `road_mask` function which yields vertices to be used by the perspective transform, as well as matrix (`mtx`) and destination (`dst`) camera calibration data, that was pickled and imported in the previous code blocks.  

A binary image is created by using the previously discussed `colour_grad_thresh` function, that yields a binary warped image:

![binarywarp][binarywarp]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Firstly I created a Line Class as shown in the eigth code block of the notebook. This line class is used to store all the relevant data for each lane, and can dynamically update the number of average previous frames to average by the use of a `refresh()` function.

To detect lane lines without prior information I followed the windowed search approach from the course, and created a `window_lanefind` function as shown in the ninth code block of the notebook. My function creates a histogram of the columns in the binary warped image, and finds the left and right lane midpoints as the maximum of the histograms. A 12 windowed search is conducted along each line with a pixel margin of 125, and requires a minimum detection of 50 pixels to shift each window midpoint as the window moves up the image. The pooled pixels in all the windows are then used by a polyfit function to generate the coefficients of a second order polynomial to fit the lane lines.

An example of the window search is shown here:

![window][window]

To continuously detect lane lines, with a slightly lower latency of the `window_lanefind` function, I created a `cont_linesearch` function as shown in the tenth code block of the notebook. This function takes in a new image and the existing Line classes, which are assumed to have previous line fit data, and a new line search is conducted within a 100 margin window of the previous best fit lines. The lines and margin for the continued line search are shown here:

![continue][continue]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculated the radius of curvature for each line in the same windowed and continued line search functions discussed previously, and shown in the ninth and tenth code block of the notebook. The radius of curvature is calculated for each line and uses an assumed real world meter to pixel ratios:

`y direction m_per_pix = 30/720, x direction m_per_pix = 3.7/700`

The center position is also calculated in the same code blocks, using the same x direction pixel ratio.

The Radius of Curvature (RoC) and Distance from Center (DfC) for each line are then applied to the image as a Heads Up Display (HUD) effect as shown in the eleventh code block of the notebook, with an example shown here:

![hud][hud]

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the twelfth code block of the notebook in the function called `lane_find_pipeline`.  Here is an example of my result on several test images:

![test1][test1]
![test2][test2]
![test3][test3]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my main test video result](my_test.mp4)

Also I applied it to the [challenge video](my_challenge_video.mp4)
and the [harder challenge video](my_harder_challenge_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Several iterations of this project were pursued before this final result, gradually building in features that made the system more robust. For example; including a Line class that averages over `n` last best fits of the lane detection, and experimenting with different margins in the windowed line search to optimize detection of highly curved lines. 

The road mask function applies a trapezoid with hard coded parameters which turned out to be problematic for bumpy roads or those with extreme curvature and hills as shown in the harder challenge video. Future work would consider finding an artificial horizon and dynamically applying the trapezoid based on an initial first pass of the image.

The generation of a `binary_warped` image using sobel threshold and s channel values were quite sensitive to the quality of the input video, specifically the lighting  of the system. As shown in the challenge videos, extremely dark regions from shadows and saturation from sunlight flaring the camera tended to throw off the lane prediction. The sanity checks and averaging logic I used works for short duration and the vehicle can likely power through any short areas of concern (like a heavy shadow under a bridge) but for the consistently changing lighting conditions of the hardest challenge video, it can be seen that the algorithm has trouble through most of the exercise and would need further refinement in the future such as horizon adjustment of the roadmask and normalizing the lighting conditions to make a better outcome.

Sanity Checks were applied for the lane lines to either reduce the number of averaging lines for fast changing curves (small radius of curvature), or to throw away detection of lane lines if they somehow got mixed up (e.g. right lane on the left of the left lane). Future work could consider more sanity checks on the lines such as comparing the radii of curvature to make sure they are somewhat equal, and also perhaps to use the radius of curvature to intelligently predict the artificial horizon and and likely future curvature of the road much like human drivers anticipate a curve continuing gradually. This information would feed back into the road mask application.

This was a fun and challenging project, but like with all things, there was not enough time to fully explore everything I wanted to.


