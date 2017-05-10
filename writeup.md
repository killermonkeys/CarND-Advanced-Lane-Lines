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

[image1]: ./output_images/example1.png "Undistorted"
[image2]: ./output_images/example2.png "Road Transformed"
[image3]: ./output_images/example3.png "Binary Example"
[image4]: ./output_images/example4.png "Fit"
[image5]: ./output_images/example5.png "Lines on road"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2nd code cell of the IPython notebook located in "./examples/example.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image1]

I used the camera calibration matrix which were output from the calibration step above, and undistorted using `cv2.undistort()` on each image. This step is part of my perspective transform `warp_lanes()` in the 5th code cell.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used two different sets of transforms to find lines. All the code is in the 6th code cell.

First, changed to the HLS color space. Then I used the two channels to find white (R and G when recombined) and one to find yellow lines (B when recombined).  

*White lines* first used a channel threshold where L > 170 and S < 40, i.e. bright areas with low saturation. Then I used Sobel x and y with a ksize=7. 

*Yellow lines* used a channel threshold where H between 20 - 40 (in OpenCV's representation this is equivalent to 40° to 80°, L and S between 30 - 240. Yellow was relatively uncommon in all but the harder challenge so it was sufficient to isolate yellow. 

You can see a sample result below. I spent quite a lot of time tweaking this to get to this stage but it's clear that in order to succeed at the hard challenge I would need to continue to improve it.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code in `warp_lanes()` in the 5th code cell does this step. First it undistorts the image, then I draw a polygon whose points I found by inspecting the images in photoshop. I drew a trapezoid on each image representing the road bounds and then averaged the values and centered it on the 50% of the image (since the assumption was that the camera was mounted on the vehicle centerline). All the points are in percentages, and each poly is multiplied by the image dimensions. These points drive a perspective transform using `cv2.getPerspectiveTransform()` and `cv2.warpPerspective()`.

I verified the points were correct by checking road stills where the roads were relatively parallel. The perspective transform kept them generally parallel, although I did notice variance between the videos, so a calibration step might have been useful here.

![alt text][image2]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for identifying centroids is in `find_window_centroids()` in the 7th code cell. I used the sliding window convolution method with several tweaks from the suggested concept. 

My code always performs the starting step (i.e. I did not use the suggestion of looking near the previous frame's lanes) of finding the bottom starting point. I limited the area to the bottom quarter and the avoided the third. This works well for highway driving but failed pretty badly on the harder challenge video. I then divided the frame into 10 layers and move up each layer (this allowed me to narrow the window) and find centers using the convolution. I had to change the convolution code to place the window center at the actual center, otherwise it was too easy to "find" a blob like the HOV diamond and jump over.

After I find the centers, I paint the pixels in each window using `draw_centroids()`. This takes the pixels in the window areas that are highlighted by the binary lane pixel detection and "adds them", resulting in a list of the pixels. 

The same code has some handling for painting the pixels on top of the existing image for diagnostics.

![alt text][image4]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

This was also in the 7th code cell.

I calculated the radius by scaling the pixel dimensions to meters, and then using the radius formula suggested. The `measure_curvature()` function does this for a single lane line.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I collected together the pipeline steps again in the 9th code cell (Full pipeline) which does each pipeline step and then draws the lines onto the original image.

![alt text][image5]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

[project_video.mp4 output](./project_video_output.mp4): successful
[challenge_video.mp4 output](./challenge_video_output.mp4): successful 
[harder_challenge_video.mp4 output](./harder_challenge_video_output.mp4): not successful

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

My final pipeline appears to be good at highway style driving where lanes are clearly painted (and exist!), well-lit, good initial conditions, relatively low curvature, and low slope. The harder_challenge_video is much much harder and nearly every step of the pipeline would need to be improved.

By working mostly on the thresholding step I was able to get very precise lanes with very few false positives. The issue is that having a conservative approach like this fails in more challenging conditions, and subsequent pipeline steps fail. I arrived at this conservative approach by initially playing with the thresholds and combinations until I took a step back and decided to focus individually on white and yellow lines. 

Expectations in my algorithm:
* Lanes are well-lit
* Lanes start out with good initial conditions (i.e. you are in the center of the lane when starting)
* Lines are white or yellow and always exist
* Contrast between the pavement and paint is good
* There is no white or yellow area near the in the peripheral areas of the image
* Curves are gentle
* Slope is minimal

As you can see this is a pretty limited set of roadways. However while looking at the harder challenge road, I believe that some things could improve the situation:
* Using the previous lanes estimate to define the window of the next frame.
* Understanding the 3D representation of the image (i.e. lines should only appear on the ground). This could be done with lidar or with stereoscopic cameras.
* More work on filtering at the image processing step
* Looking at curved or straight Hough transforms
* Handling shadows better, possibly with illumination, or by identifying shadow areas and performing different image processing in them
* Combining with ML approach?

Overall for a naive implementation, the ML approach is much better performing! I strongly suspect I could project lane lines 