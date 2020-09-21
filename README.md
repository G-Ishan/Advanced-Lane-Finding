## Writeup 
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

[image1]: ./output_images/found_corners.jpg "Corners"
[image2]: ./output_images/undistorted_checker.jpg "Undistorted Checker"
[image3]: ./test_images/straight_lines1.jpg "Straight Lines"
[image4]: ./output_images/undistorted.jpg "Undistorted"
[image5]: ./output_images/s_channel_binary.png "S Channel"
[image6]: ./output_images/combined_thresh.png "Combined"
[image7]: ./output_images/roi_coordinates.png "ROI"
[image8]: ./output_images/warped.jpg "Warped"
[image9]: ./output_images/warped_lanelines.png "Warped Visual"
[image10]: ./output_images/binary_warped.png "Binary Warped"
[image11]: ./output_images/sliding_windows.jpg "Sliding Windows"
[image12]: ./output_images/search_around_poly.jpg "Prev. Info"
[image13]: ./output_images/road_info.png "Road Info"

[video1]: ./output_videos/project_video_output.mp4 "Output Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup 


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The first step before processing any images of the road was to calibrate the camera. This was done by importing calibration images of chessboards. Assuming that the chessboard is fixed on the wall, I updated `objpoints` and `imgpoints` whenever there was a successful detection of the chessboard corners in the calibration image. I then used `objpoints` and `imgpoints` to compute the camera calibration matirx and distortion coefficients using the openCV. The distortion coefficients were used to undistort the calibration images. The images below show an example of corner finding and undistortion.

![alt text][image1]
![alt text][image2]

### Pipeline (single images)

To demonstrate my pipeline, I will describe how each step of the pipeline is applied on a simple test image where the road lines are fairly straight like this one:

![alt text][image3]

For ALL the steps below: the jupyter notebook `advanced_lane_finding.ipynb` contains headings to separate the code that applies to each step in the rubric. This not only makes the code more readable but is how I tackled the problem - one step at a time.

#### 1. Provide an example of a distortion-corrected image.

The distortion correction is applied to the image:
![alt text][image4]

#### 2. Describe how you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image. I first applied an HLS color filter and isolated the S channel. I also applied a sobel operator and then combined these thresholdings to get an optimal result. Below you can see an example of the isolated and combined thresholding.

![alt text][image5]
![alt_text][image6]

#### 3. Describe how you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp_image()`.  This function takes as inputs an image (`image`), as well as source (`src`) and destination (`dst`) points.  I hardcoded the source and destination points and chose the points based on where lane lines are likely to be shown in then image. It was a little bit of visual guess and check. I drew over an image to confirm where the lane lines were:

![alt_text][image7]

I then warped the image:

![alt_text][image8]

To verify visually that my ROI was where I wanted and wrapping the lane lines correctly. I drew the ROI lines onto the original image and the warped image:

![alt text][image9]

I then applied the thresholding from step 2 to the warped image to get a binary warped image that will be used for detection of the lane lines:

![alt_text][image10]

#### 4. Describe how you identified lane-line pixels and fit their positions with a polynomial?

I then implemented two functions - `find_intial_lane_pixels()` and `search_around_prev_frame()`. The first is meant to be used to find lane pixels in then image if doing a blind search - meaning no lane data is previously available. This function applies a histogram to my binary warped image in order to find the base of the lane lines. Once the base of the lines are found, I implemented a method of sliding windows that searches for lane pixels in a defined window region and works its way up the image. Once the lane lines are detected, a second order polynomial is fit to the lines to define the lane line. 

![alt text][image11]

However, if we know calculate the lane lines for the current frame, we do not need to search blindly for the next frame. Therefore, the `search_around_prev_frame()` uses the polynomial fit from the previous frame to only search for pixels around the area of the known fit. I set a window margin of 100 pixels, and also implemented a smoothing function that averages the information from the last 5 frames and got the following result:

![alt_text][image12]

#### 5. Describe how you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Once the lane lines were found and I fitted polynomials to each line, I calculated the radius of curvature for each line. To measure the curvature, I had to transform then pixels we see in the image to real world measurements. To do this, I used U.S. regulations for lane lines (30 meters long and 3.7 meters wide). We know that our camera image will have 720 relevant pixels in the y-dimension and roughly 700 pixels in the x-dimension, so I was able to get a conversion for the y and x measurements in meters per pixel in the image. Then, because I know the polynomial fit for each lane line, I can get the radius of curvature.

Given a polynomial f(y) = Ay^2 + By + C and
R_Curve = (1+f'(y)^2)^(3/2) / f''(y) 

I can calcualte the radius of curvature using the second derivative of the polynomial and my known x and y values.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Now that I know my radius of curvature and lane line positions, I used `draw_lane()` and `disp_road_info()` to overlay the lane and the important road information onto the image. To wrap the lane back onto the image I had to get the inverse distortion coefficients for the iamge and then unwarp the image. The result below shows and example of the final result:


![alt text][image13]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my output video](./output_videos/project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

At first, I worked through my pipeline without implementing a Line class and simply doing a blind search. Not only was the processing time large, but the pipeline struggled during curves. After implementing the Line class and using previous information, the lane output is much smoother. Moreover, my biggest struggle when implementing the project was getting lost in the polynomial code. I was able to solve the overall problem efficiently and did not have an issue with programming my solution but for the polynomial fits specifically, it is a lot of math with similar variable names so I was not assigning the right variables sometimes. This was especially true when using previous frame information. Once I wrote it out on pen and paper I was able to understand it better.

In terms of improvements, I think my pipeline would likely fail in areas of a lot of shadows and/or roads with quick sharp turns. I could make this function more robust by including sanity checks for differences in lane lines from frame to frame. I have an attribute in my Line class to calculate the difference between then current line fit to the previous but I didn't actually use it for anything useful. I could make sure that if a shadow makes the lane line jump that a difference of some threshold is ignored and the previous lane line is used. Moreover, the previous frame could be completely ignored for sharp curves and it might be beneficial to use a blind search if the lane changes drastically. 

Because of time constraints due to a full time job, I decided to not implement this optimizations at this time and to tackle the challenge videos at a future date.
