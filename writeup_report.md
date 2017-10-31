
# Advanced Lane Finding Project

### The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[camera_calibration]: ./output_images/camera_calibration.png "Camera calibration"
[distortion]: ./output_images/distortion.png "Undistortion examples"
[threshold]: ./output_images/threshold.png "Threshold examples"
[perspective]: ./output_images/perspective.png "Perspective transform examples"
[color_fit_lines]: ./examples/color_fit_lines.jpg "Equations"
[pipeline]: ./output_images/pipeline.png "Pipeline"
[second_approach]: ./output_images/second_approach.png "Second approach"


## [Rubric Points](https://review.udacity.com/#!/rubrics/571/view)

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.

The present document is the writeup report

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./CarND-Advanced-Lane-Lines.ipynb" 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][camera_calibration]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

On the image below I show the distortion correction applied to each of the test images:

![alt text][distortion]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Instead of using gradients as suggested in the classroom, I only did color thresholding to get the white and yellow lines. This approach gave me much better results, specially for the challenge video.

But first I did a histogram equalization for each channel of the RGB image, as suggested [in this article](https://medium.com/@matlihan/how-to-overcome-severe-shadows-in-a-lane-finding-project-for-autonomous-vehicles-8f605be8b766), in order to decrease the effect of different lighting conditions.

Then I converted the image to the HLS colorspace, filtered the L channel values between 235 and 255 for the white lines, whereas for the yellow line, I used only values between 100 and 250 for the S channel and between 10 and 22 for the H channel.

The code is contained in the code cell #5 and the resulting binary versions of the test images are shown elow:

![alt text][threshold]

All those parameters were empirically tunned.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform can be found in the code cell #7. I chose the source points by looking at the two test images that appeared to be quite straight, and manually selecting those points. Then I chose the destination points in such a way that the image would be horizontaly centered and any point outside an area of interest would discarded. 

This resulted in the following source and destination points that were hard coded into this function:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 521, 498      | 290, 500      | 
| 759, 498      | 990, 500      |
| 271, 665      | 290, 720      |
| 1009, 665     | 990, 720      |

This function has also an inverse parameter, so the lane detected could be drawn back to the original image, by inverting the source and destination points.

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][perspective]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I use two different approaches to find the lane-line pixels, and this can be found on the code cell #12.

In the first approach I use the peaks of the histogram of the bottom half of the image, in order to guess where the lane lines are. Then I divide the height of the image in 9 windows 200 pixels wide for each lane line, starting from bottom up, horizontaly centering the first windows on the peaks of the histogram, then centering the next window in the median of the non-zero pixels of the last window. All the non-zero pixels within all windows are considered to be part of the lane-lines

Then, when I have detected a lane, I switch to another approach which consists of, for each side, search for non-zero pixels within a band of 200 pixels centered on the last detecded line.

In order to handle the lane-line operations I created the class `Line` on code cell #10 to be used for each lane-line and the class `Lane` on code cell #11 with has an instance of the class `Line` for each line.

After the lane-line pixels have been identified, they are passed to the method `set_new_points` of the class `Line` which uses the `numpy` function `polyfit` to find coeficients of the second order polynomial which will represent the correspondent lane-line.

With the coeficients for the polynomial of each lane-line, the lines are calculated as shown below:

![alt text][color_fit_lines]

Then, the method `sanity` of the class `Lane` is called to check whether the calculated lane-lines are consistent or not. If so, an average of the last 10 lane-lines are considered for the next iterations, if not, the last consistent lane-lines are replicated and the average is taken. If there are 10 consecutive inconsistent lane-line detections, every thing is discarted and the process starts all over again with the first approach.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In order to calculate the radius of curvature and the position of the vehicle, first it is necessary to convert the pixels to meters. To do so, I defined the constants `YM_PER_PIX` of 30 meters per 700 pixels in the y direction and `XM_PER_PIX` of 3.7 meters per 700 pixels in the x direction. (code cell #9)

For each lane-line, the radius was calculated on the `compute_radius` method of the `Line` class (code cell #10) by converting the x and y points to meters, repeating the polyfit calculation and, with the resulting coeficients, the radius is equal to the expression: `((1 + (2*A*y*YM_PER_PIX + B)**2)**1.5) / abs(2*A)`. For the next steps the average of the radius of the last 10 valid lane-lines are considered.

Finally, in the `compute_data` method of the `Lane` class (code cell #11), the left line radius and the right line radius are averaged to get the lane radius. If the radius is greater then 3000 meters, the lane is considered to be straight.

Also in the `compute_data` method, the center of the image is considered to be the center of the car, so the difference of the center of the image to the center of the lane (average of the x position of each line on the bottom of the image), converted to meters is the position of the vehicle with respect to center.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The function in the code cell #12 also outputs an image containing the detected lane, and opitionally the windows or line bands in which were considered the non-zero pixels to be part of the lane-lines and this image can be plotted back down onto the road.

The windows are plotted as pink squares, the line bands are pink stripes, and the detected lane is in green when there is a consistent detection of the lane-line, and red when there isn't, therefore an average of the last detection is being considered. Then an inverse perspective transform must be applied to that image, which is added to the original with the function described at the code cell #13.

The whole pipeline is described at the code cell #14 and here is an example of it applied to each test image:

![alt text][pipeline]

To illustrate how the second approach described above improves the line detection, lets compare the third exemple from the image above, with its corresponding frame from the resulting video:

![alt text][second_approach]



---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://youtu.be/EN7TFCGj2BI)

And here's a [link to my challenge video result](https://youtu.be/aZ44rgm_33E)


---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?


I found this project really exciting, but sadly I've been through a tough moment at my job and had little time to work on it.

Shadows and different lighting conditions is still very hard to deal with, at least the tracking tecniques used were enough for the project video and the challenge video. Unfortunately I didn't have time to take the harder challenge, which seems to really push it hard with so many tight curves, shadows and passing vehicles.

The mounting of the camera on the challenge video, seemed to be different from the project video, or perhaps just the slope of the road was different. This, plus the different light and road conditions, made it very hard to generalize one solution to both videos, so I had to do different tunning for the thresholds and perspective parameters. So I guess it is useless in a real life situation. I thing there should be some more dynamic approach to do this task, maybe combining computer vision with machine learning or other classification tecnique.

I really hope to go further on this problem someday.



```python

```
