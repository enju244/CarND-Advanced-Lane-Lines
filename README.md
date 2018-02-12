## Advanced Lane Finding

[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)

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

[image1]: ./images/chess1.png
[image2]: ./images/chess2.png
[image3]: ./images/undist_pre.jpg
[image4]: ./images/undist_post.png
[image5]: ./images/bin_stack.png
[image6]: ./images/bin_thresh.png
[image7]: ./images/pt_str_pre.png
[image8]: ./images/pt_str_post.png
[image9]: ./images/pt_cur_pre.png
[imageA]: ./images/pt_cur_post.png
[imageB]: ./images/window1.png
[imageC]: ./images/window2.png
[imageD]: ./images/polyfit1.png
[imageE]: ./images/polyfit2.png
[imageF]: ./images/outsample.png


---


Camera Calibration
---

Image distortion occurs when the camera transforms 3D objects in the real world into 2D images. It is important to correct for image distortion so that we have an accurate representation of the appearance of objects in images. Chessboards are used in calibration, since it's regular and high-contrast pattern makes them easy to detect.

The code for this step is contained in P4.ipynb, in the cells under the "Camera Calibration" section. The camera matrix and distortion coefficients are prepared as follows:

- Prepare arrays for maintaining object points and image points:
 
 - Object points, which are represented as a 3D tuple (x, y, z) representing a coordinate of one of the inner chessboard corners. For this project, we use a chessboard with 9x6 inner corners.

 - A corresponding set of image points stores the position of the points in the image that correspond to the chessboard corners.

- [cv2.findChessboardCorners](https://docs.opencv.org/2.4/modules/calib3d/doc/camera_calibration_and_3d_reconstruction.html#bool%20findChessboardCorners\(InputArray%20image,%20Size%20patternSize,%20OutputArray%20corners,%20int%20flags\))is used to find the corners of the chessboard in the given image.

- [cv2.calibrateCamera](https://docs.opencv.org/2.4/modules/calib3d/doc/camera_calibration_and_3d_reconstruction.html#double%20calibrateCamera\(InputArrayOfArrays%20objectPoints,%20InputArrayOfArrays%20imagePoints,%20Size%20imageSize,%20InputOutputArray%20cameraMatrix,%20InputOutputArray%20distCoeffs,%20OutputArrayOfArrays%20rvecs,%20OutputArrayOfArrays%20tvecs,%20int%20flags,%20TermCriteria%20criteria\)) is used to obtain the distortion coefficients, based on the collected object points and image points. Then, using the distortion coefficients, [cv2.undistort](https://docs.opencv.org/2.4/modules/imgproc/doc/geometric_transformations.html#void%20undistort\(InputArray%20src,%20OutputArray%20dst,%20InputArray%20cameraMatrix,%20InputArray%20distCoeffs,%20InputArray%20newCameraMatrix\)) is used to apply un-distortion on the image.

- Finally, [cv2.getPerspectiveTransform](https://docs.opencv.org/2.4/modules/imgproc/doc/geometric_transformations.html#Mat%20getPerspectiveTransform\(InputArray%20src,%20InputArray%20dst\)) is used to obtain the transform matrix for a given src and dst. Src coordinates are chosen to be the four outer corners of all of the chessboard corners found during the undistortion step.

This process is repeated for the various calibration images. Here are some samples images of the undistortion process. 

![ch1][image1]
![ch2][image2]



Pipeline (Single Images)
---

The code for this section is contained in P4.ipynb, in the cells under the "Pipeline (test images)" section. 


#### Distortion Correction

The following illustrates distortion correction being applied to one of the frames from the project video. 

![pre][image3]
![post][image4]

Note that the objects closer to the edges of the image (or closer to the outer edges of the camera lens), such as the white car on the right side of the image, have been "spread out" accordingly to reflect the actual distance to them. The undistortion was done using the cv2.undistort with the distortion coefficients obtained from the camera calibration step.



#### Color Transforms and Gradients for Thresholding

Various methods have been applied to obtain a binary threshold image that can identify locations of lane lines.

- _Sobel Operator_ for gradients in the X direction detect the lane lines well, and were used as part of the thresholding.

- _Red (R)_ channel from RGB representation of image does well in detecting the white colors of the lanes. 

- _Saturation (S)_ and _Hue (H)_ channels from _HLS color space_ representation of image. The Saturation channel is useful for identifying the bright colors, and in particular the yellow lines. Low thresholding was applied on Hue for white lines as well, to reinforce the work done by Red channel.


Below is a stacked image of the Sobel, Red, and Saturation binary thresholding results to create a combined binary image.

![comb][image5]


Of course, the binary threshold image is not colored, but rather takes into account which pixels are High vs Low. Following is the corresponding binary image.

![bin][image6]



#### Perspective Transform

The src and dst for the perspective transform were calculated by eye-balling the trapezoidal vertices corresponding to the lane on the road if the car were on a straightaway. [cv2.getPerspectiveTransform](https://docs.opencv.org/2.4/modules/imgproc/doc/geometric_transformations.html#Mat%20getPerspectiveTransform\(InputArray%20src,%20InputArray%20dst\)) was used to obtain a transformation (and the inverse) matrix. This was followed by [cv2.warpperspective](https://docs.opencv.org/2.4/modules/imgproc/doc/geometric_transformations.html#warpperspective) to apply the trasformation matrix to the frame image to obtain a 'birds-eye view' of the segment of the lane right ahead of the vehicle. 


Below is an example of applying the perspective transform on a straight segment of the road:

![str_pre][image7]

![str_post][image8]


In addition, here is an example of applying the perspective transform on a curved segment of the road:

![cur_pre][image9]

![cur_post][imageA]


It can be seen that the transformation is relatively good, in the sense that the lanes are parallel, and that lanes deviate from the rectangular portion of the transformed image in the case of curved lanes. 



#### Lane-Line Pixels and Polynomial Fitting

Lane lines were detected using the sliding window histogram search. This approach takes horizontal segments of the perspective transformed binary image, and takes a histogram across columns to identify peaks in which there is a higher concentration of "High" pixels. This is done for both the left and right half of the image (for the left and right lanes, and then is repeated for varyous horizontal segments of the image. 

Below are example visualizations of the detected lane points after the sliding window method.

![win1][imageB]

![win2][imageC]

Finally, np.polyfit was used to fit a second order polynomial through the points detected using the sliding window. Below are corresponding extrapolations of the polynomial onto the points detected by the sliding window method.

![poly1][imageD]

![poly2][imageE]




#### Radius of Curvature and Vehicle Position

The radius of the curvature of the road is calculated using the polynomial coefficients (A, B, C) as follows:

This however, calculates the curvature in based on pixel values. This is modified by applying a conversion in x and y from pixels space to meters in real world space. The code for this is located around line ### in the notebook cell for "Pipeline (test images)."

Qualitatively, the radius of curvature serves as a good sanity check, especially when debugging. For curved lanes, the radius on the corresponding side should be smaller compared to when the car is on a straightaway.

Similarly, the offset of the vehicle from the center is calculated using the relative location of the bottommost points corresponding to the two edges of the detected lane line and the center of the image (where the camera is assumed to be located). The midpoint of the edge points is compared to the center of the image, and then is converted from pixel to meters. 

The values of the radius and the center offset are not presented in the report, since it was used more as a qualitative tool for debugging, vs an accurate measure. However, the particular values for the test images will be printed (along with the rest of the images) by running the cell for "Pipeline (test images)." In addition, the video pipeline output will have these measurements embedded in the video itself. 



#### Proper Lane Area Identification

Once the polynomial curves are found, we obtain the corresponding edges of the shape formed by the curves, and warped back into the original perspective using the inverse matrix. The shape is then overlayed back onto the original (undistorted) image to be used in the video. The following is an example of the final output.

![out][imageF]




Pipeline (Video)
---

Here's a [link to my video result](./project_video_output.mp4). The pipeline performs reasonably well on the entire project video. The code for this step is contained in P4.ipynb, in the cells under the "Pipeline (Video)" section. 

In addition, the video pipeline incorporates tracking of the previous curvature fits for better search. This is also elaborated in the "Discussion" Section. 



Discussion
---

### Approaches that worked 

#### Color Space Thresholding

- Thresholding the HLS S channel was very effective in detecting the yellow lines, which was somewhat difficult to detect back in project 1. 

- The RGB R channel was good at detecting white lines. Taking the HLS H channel reinforced this by adding slightly better detection of white lanes in areas covered by shadows. 

#### Tracking/Smoothing using N previous fits

Re-using the polynomial fit used in the previous frame was implemented in the video pipeline. Taking this average of the N previous polynomial fit was:

- _logical_, because the curvature of the road changes gradually across frames.
- _practical_, since it saves computation time of having to recalculate via sliding window.
- _(slightly) robust_, since the previous N fits will reject outliers in the case a bad estimate of the curvature is obtained. 


### Areas for improvement

#### Thresholding

A better search in the thresholding parameters can be achieved with more exploration. A potential issue is with the generalizability of such parameter search. For example, difference in lighting in the other challenge videos, may lead to inconsistent detection, in which case we'd need some kind of adaptive thresholding. 

#### Perspective Transform

For this implementation, I kept the values that seemed to work for the project video. However, I think this can be improved since there is small distortion towards the top of the images. An improvement here may also improve the polynomial fitting as well.

#### Robustness

The sanity check across frames can be improved by incorporating more available data, such as rejecting fits that have unreasonable radius of curvatures, and checking horizontal distance between lanes.



Description of Output Images 
---

There are two sets of images, for inputs test1 and test2. The filenames are formatted as [input_file]_[Number]_[Description].png. The summary of description are as follows:

- test#_01_undist: Image after correcting for distortions.
- test#_02_bin_stack: The visualization of contributions of binary thresholding on Sobel, Red channel, and Saturation channel towards a binary threshold image.
- test#_03_bin_thresh: The actual binary threshold image.
- test#_04_roi_overlay_[binary/color]: A visualization of the src region used for the perspective transform. 
- test#_05_perspectiveT_[binary/color]: Image after applying perspective transformation.
- test#_06_undist: Image of lane pixels identified via sliding window search.
- test#_07_polyfit: Image containing extrapolation of fitted polynomial curve.
- test#_all_in_one: Image containing the previous steps, side-by-side, for comparison.
- test#_output: The output image of the pipeline containing an unwarped image of the frame with the lane lines annotated in light green color.


