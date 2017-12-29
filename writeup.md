## Writeup Template

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

[image1]: ./images/calibration2.png "Original"
[image2]: ./images/calibration2_undistorted.png "Undistorted"
[image3]: ./images/distortion-corrected.png "Distortion Corrected"
[image4]: ./images/binary_straight_lines1.png "Binary Image"
[image5]: ./images/plot_dst.png "Plot dst"
[image6]: ./images/plot_warped.png "Warped dst"
[image7]: ./images/plot_boxes.png "Draw boxes"
[image8]: ./images/plot_lane_lines.png "Draw lane lines"
[image9]: ./images/finalresult.png "Final Reesult"

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the Camera class.

1. Prepare object points
2. Create arrays to store object points (3D) and image point (2D)
3. Read chessboard images for calibration
4. For each image
    1. Convert to grayscale
    2. Call findChessboardCorners to get corners
    3. Save the object points and corners
5. Call calibrateCamera to get the camera matrix and distortion coefficients

The following is a sample input and output:

Original Image

![alt text][image1]

Undistorted Image

![alt text][image2]


### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Distortion corrected image

![alt text][image3]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used both color transform and sobel x gradient. The function names are **filtered_with_color** and **filtered_with_sx** respectively.

For both functions, HSL color transform was used. I used the following filters for HSL:

**S channel**:

| Min | Max | 
|:---:|:---:| 
| 30  | 255 |

If the side either side (50 pixels) of the image contains certain amount of noise (left 9200 and right 2400), change the threshold to:

| Min | Max | 
|:---:|:---:| 
| 100 | 255 | 

**L channel**:

| Min | Max | 
|:---:|:---:| 
| 200 | 255 |


**filtered_with_color** :

I used HSV on top of the HSL filters described above.

For HSV, I used the following threshold:

| Min     | Max         | 
|:-------:|:-----------:| 
| 0,0,170 | 255,255,255 |

For filtering out noise in the gray image, the following threshold is used:

| Min | Max | 
|:---:|:---:| 
| 10  | 255 |

If sum of the pixels from 5 to 10 is more then 2000 (lots of noise), change the threshold to the following:

| Min | Max | 
|:---:|:---:| 
| 10  | 155 |

**filtered_with_sx** :

If **filtered_with_color** did not produce a clean outline of lane lines, use Sobel x on top of the HSL filters described above. I used Sobel x with the following threshold:

| Min | Max | 
|:---:|:---:| 
| 30  | 155 |

Here's a sample binary image produced by **filtered_with_color**

![alt text][image4]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The snippet of code is from the function **warpImage**

```python
    pt1 = ((img_size[0] / 2) - 55, img_size[1] / 2 + 100)
    pt2 = ((img_size[0] / 6) - 10, img_size[1])
    pt3 = ((img_size[0] * 4 / 6) + 80, img_size[1])
    pt4 = (img_size[0] / 2 + 50, img_size[1] / 2 + 100)
    src = np.float32(
        [[pt1[0], pt1[1]],
        [pt2[0], pt2[1]],
        [pt3[0], pt3[1]],
        [pt4[0], pt4[1]]])
    dst = np.float32(
        [[(img_size[0] / 4) - 100, -500],
        [(img_size[0] / 4) - 100, img_size[1]],
        [(img_size[0] * 3 / 4), img_size[1]],
        [(img_size[0] * 3 / 4), -500]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 220, -500     | 
| 203, 720      | 220, 720      |
| 933, 720      | 960, 720      |
| 690, 460      | 960, -500     |

The blue lines look parallel, so it appears the the perspective transform is working

Undistored with points plotted

![alt text][image5]

Warped result with points plotted

![alt text][image6]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

In the function **calibrate**, I use largest value of histogram to find the initial position of the window. Then find the position of the next window by averaging the index position of the current window. The follow image is the final result.

![alt text][image7]

All x and y coordinates in the (9) windows are collected. Next we use numpy's polyfit function to find A, B, C in the equation Ax^2 + Bx + C. The follow image is the quadratic line

![alt text][image8]


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius is calculated in the function **calculateRadius** and the center offset **calculateCenterOffset**

For the radius, I converted the x and y points to meters, do a polyfit again, and calculate the radius again using the x and y in meters.

```python
ym_per_pix = self.ym_per_pix
xm_per_pix = self.xm_per_pix

# Fit new polynomials to x,y in world space
left_fit_cr = np.polyfit(self.ploty*ym_per_pix, left_x_values*xm_per_pix, 2)
right_fit_cr = np.polyfit(self.ploty*ym_per_pix, right_x_values*xm_per_pix, 2)

# Calculate the new radii of curvature
left_curverad = ((1 + (2*left_fit_cr[0]*y_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
right_curverad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])    
```

For the center offset, I calculate the midpoint of the lanes by using the last value from the polynomial equation of left and right lane, then the midpoint of the width of the image, and finally take the absolute value of the difference between the lanes' midpoint and image's midpoint.

```python
lane_center_x = ((right_fit_values[-1] + left_fit_values[-1])/2.0)*self.xm_per_pix
midpoint_x = (self.img.shape[1]/2.0) * self.xm_per_pix

center_offset = np.abs(lane_center_x - midpoint_x)
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The function **finalResult** plot the result back down onto the road. Here's the final result

![alt text][image9]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to project video](./videoOutput/project_video.mp4)

Here's a [link to my challenge video](./videoOutput/challenge_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

To make the detection more robust, I did the following:
1. Check if **filtered_with_color** using previous frame did a reasonable job detecting lanes
2. If not, try **filtered_with_sx** using previous frame
3. If not, try **filtered_with_color** again but call **calibrate**
4. If not, try **filtered_with_sx** with **calibrate**
5. If all fails, skip frame, use previous frame

Challenges:
1. I had to hardcode the values for radius and lane distances to reject
2. I had a hard time integrating the harder challenge code into this code

Likely failure:
1. Steeper turns like in harder challenge
2. Colors outside of the hardcoded filters

More robust:
1. Figure out how to detect yellow and white lines regardless of lighting
2. When the right lane is undetectable, figure out how to derive a right lane based on left lane
