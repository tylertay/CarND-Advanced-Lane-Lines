#Advanced Lane Finding Project

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

The codes for every step are all in LaneFinding.ipynb

[//]: # (Image References)

[image1]: ./output_images/undistort_output.jpg "Undistorted"
[image2]: ./output_images/undistort_image.jpg "Road Transformed"
[image3]: ./output_images/binary_combo_example.jpg "Binary Example"
[image4]: ./output_images/warped_straight_lines.jpg "Warp Example"
[image5]: ./output_images/lines_boundary.jpg "Fit Visual"
[image6]: ./output_images/example_output.jpg "Output"
[video1]: ./project_video_out.mp4 "Video"

---
###Camera Calibration

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

###Pipeline (single images)

####1. Undistort image with distortion matrix

![alt text][image2]

####2. Generate thresholded binary image by using color and gradient thresholds

![alt text][image3]

####3. Perform perspective transform

First, I use canny edge detection and hough transform to get the left and right lane lines. I then take the 4 points (start and end point of each lane lines) as my source point and then map it to destination points so that the 2 lines are parallel. Using the source and destination points, I calculate both the matrix to warp and unwarp the image which will be used later. I verified that my perspective transform was working ass expected by drawing the source and destination points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

After the image is being undistorted and warped, I used color and gradient threshold to get the histogram of the pixels in the x-axis. I then slide 2 windows with fixed width and fixed width apart from one another to find the 2 peaks in the histogram. The pixels in the left and right window will be classified as left and right lane line respectively.

![alt text][image5]

####5. Calculate radius of curvature of the lane and position of the vehicle with respect to center.

From the thresholded image, I retrieve the x and y values within the left and right boundaries. 
```python
xvals_left, yvals_left, xvals_right, yvals_right = mask_lane(thresh_image, left_range, right_range)
```
Since the lane is about 30 meters long and 3.7 meters wide, I use that to estimate the meters per pixel in the x and y direction.

    ym_per_pix = 30/720
    xm_per_pix = 3.7/880

The radius of the curvature is then calculated as follows.

    left_fit_cr = np.polyfit(yvals_left*ym_per_pix, xvals_left*xm_per_pix, 2)
    right_fit_cr = np.polyfit(yvals_right*ym_per_pix, xvals_right*xm_per_pix, 2)
    y_eval = np.max(yvals_right)
    left_curverad = ((1 + (2*left_fit_cr[0]*y_eval + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
    y_eval = np.max(yvals_right)
    right_curverad = ((1 + (2*right_fit_cr[0]*y_eval + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])
    average_curverad = (left_curverad+right_curverad)/2

Since the lane lines are projected to 200 and 1080 on the x-axis for left and right lane respectively, we use that as reference to measure how far the vehicle is to the right or left from the center. I take the bottom x points for the left and right lane and calculate how much they shift from the center and then average them.

    offset = (xvals_left[0] - 200 + xvals_right[0] - 1080) / 2
    if offset < 0:
        offset_meter = xm_per_pix * np.abs(offset)
        cv2.putText(res_image,'Vehicle is ' + str(round(offset_meter,2)) + 'm left of center', (25,100), cv2.FONT_HERSHEY_SIMPLEX, 1.5, (255,255,255), 2)
    elif offset > 0:
        offset_meter = xm_per_pix * offset
        cv2.putText(res_image,'Vehicle is ' + str(round(offset_meter,2)) + 'm right of center', (25,100), cv2.FONT_HERSHEY_SIMPLEX, 1.5, (255,255,255), 2)
    else:
        cv2.putText(res_image,'Vehicle is in the center', (25,100), cv2.FONT_HERSHEY_SIMPLEX, 1.5, (255,255,255), 2)

####6. Example result

![alt text][image6]

---

###Pipeline (video)

Here's a [link to my video result][video1]

---

###Discussion

For this project, I did a lot of parameter tuning especially the threshold values. While tuning the parameter, it seem to work well for some images while changing its value will break for other images. So everytime I change a parameter value, I have to check that it works well for all the special cases (shadows, different ground color). My pipeline will likely fail when there are signs in the middle of the lane or when it is raining. It will also fail if there is a vehicle blocking at the front or when changing lane. To make it more robust, I should try it on more videos and use different approach of tracking the lane lines for different scenarios. There should also be vehicle detection or lane changing detection. Lastly, the resulting pipeline does not take into account of previous frames as it did not turn out to have better performance. In the future, I will try different methods of incorporating information from previous frames to get better performance.