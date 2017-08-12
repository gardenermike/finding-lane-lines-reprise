
## Advanced Lane Finding Project

The [IPython/jupyter notebook](https://github.com/gardenermike/finding-lane-lines-reprise/blob/master/lane_lines.ipynb) in this repo implements a heuristic-based approach to finding lane lines in images or video.
The basic algorithm is:
* Apply camera distortion correction to the raw images
* Find regions (i.e. lines) of strong gradient in X (using a sobel filter)
* Finding regions (again, hopefully lane lines) of the image with strong saturation
* Combining the gradient and saturation-filtered images and masking away regions unlikely to hold lane lines
* Perspective transforming the image to an overhead perspective to make parallel line finding easier
* Applying a series of window to the resulting image to find the centers of greatest pixel intensity
* Finding a second-degree polynomial to best fit the centroid points found above
* Drawing lines and filling the space between them to cover the lane area
* Perspective transforming the resulting image back onto the original image
* Measuring approximate distance between lane lines and radius of lane curvature to perform sanity checks and smoothing on video

[//]: # (Image References)

[undistorted]: ./examples/undistorted.jpg "Undistorted"
[centroids]: ./examples/centroids.png "Centroids"
[uncalibrated chessboard]: ./camera_cal/calibration1.jpg "Uncalibrated chssboard"
[chessboard calibrated]: ./examples/chessboard_calibrated.png "Calibrated chessboard"
[base image]: ./test_images/test2.jpg "Unprocessed highway image"
[filtered combined masked]: ./examples/filtered_image_combined_masked.png "Flattened, single-channel filtered and masked image"
[filtered three channels]: ./examples/filtered_image_three_channels.png "Filtered three-channel image"
[overlay]: ./examples/overlay.png "Highway with overlay"
[perspective transformed]: ./examples/perspective_transformed.png "Perspective transformed highway"
[preprocessed]: ./examples/preprocessed.png "Fully preprocessed image ready for lane detection"

---

### Camera Calibration


The code for this step is contained in the second code cell code cell of the [IPython notebook](https://github.com/gardenermike/finding-lane-lines-reprise/blob/master/lane_lines.ipynb).

I used the [OpenCV](http://opencv.org/) library to calibrate the camera used for the images and video. OpenCV provides a `findChessboardCorners` function to detect the corners of a chessboard in an image. Since chessboards have a regular pattern with clear boundaries, they are perfect for finding distortion in a camera image. I used a series of 20 calibration images taken with the camera used to generate the source images. The calibration images are of a chessboard in various positions in thefield of view. I use `findChessboardCorners` to find the (x, y) coordinates of the corners in each image. I also generate a set of perfectly consistent corners (`objp` in the code) on an imaginary flat plane in front of the camera.
I use OpenCV's `calibrateCamera` function to map the warped real-world camera points to the mathematically perfect `objp` points, resulting in a matrix and coefficients of distortion that can be passed to OpenCV's `undistort` method to remove camera distortion. The result is a simple method (`undistort`) to undistort any image taken with the camer. An uncalibrated and calibrated image are below.

![Calibrated chessboard][chessboard calibrated]

The IPython notebook shows each calibration image and its undistorted pair.


### Pipeline (single images)

I applied a series of transformations to each image to find the lane lines.

#### 1. Undistortion
For each image, I removed camera distortion using the aforementioned `undistort` method. An example base image and undistorted image are below. Note that without the visual reference of straight lines like those found in the chessboard, it is difficult to see a difference. However, after perspective shifting, the differences are exacerbated, making the unwarping more useful.

![Base image with distortion][base image]
![Image with distortion removed][undistorted]


#### 2. Filters

In the third code cell in the notebook (the `extract_features` function), I apply a [Sobel filter](https://en.wikipedia.org/wiki/Sobel_operator) in the horizontal (x) dimension.  A Sobel filter is approximates the gradient at each point. Since the lane lines are more vertical than horizontal, I am very interested in sudden changes (i.e. a high gradient) in the x dimension. A sudden change in x would suggest a bright vertical object, which is likely to be a lane line!
I first converted the image from RGB to an [HSL color space](https://en.wikipedia.org/wiki/HSL_and_HSV). Since lane lines are intended to have bright color, checking for lightness and saturation is more useful than grayscale or RGB. I applied the sobel filter to both the S and L channels.

A problem with Sobel filtering that was made clear in the second video is that long joints in the road may have a stronger gradient signal than the lane lines themselves. Again, in a production system, I would probably use an ensemble of inputs, and fall back to sobel filtering if the color-based detection was insufficient,

My primary source of lane line information was applying a threshold to the S channel. Lane lines are, by design, painted with bright colors, so the S channel was a good indicator for both yellow and white lines. A saturation threshold achieved better results in all videos I tried. However, it had its own problems: color tends to wash out at distance, so the signal was insufficiently strong. I found that for the primary video, thresholding the filter at 170 cut out most noise while leaving a strong lane line signal. However, I also found that on other videos the signal was not strong enough at that threshold. For a production system, I would experiment with adjusting the threshold to match the maximum level in the image, in order to balance signal vs noise.

To get the strongest signal, I used a binary OR to combine hit pixels from the Sobel filters in S and L and the color threshold in S. I got an excellent signal with the combined values. I the image below, I show an original image, the three filters mapped to RGB to see their overlap and respective strengths, and the combined filtered image. Note in particular the yellow lines. The yellow is where the two sobel filters intersect, which is almost everywhere that either of them have signal. Leaving out one of them would lose very little information. The blue-only lines are where the saturation provided the only signal. The saturation is nearly sufficient on its own.

![Image filtering details][filtered combined masked]

I also masked out the top of the image, as well as triangles on the left and right and one down the center, in order to remove as much noise as possible.

#### 3. Perspective transformation

The next step in the pipeline was to transform the image to approximately an overhead perspective. The `perspective_transform` method does the work.
To transform, I used OpenCV's `getPerspectiveTransform` function, which returns a matrix used to transform an image to a new set of points. I kept the base of the image fixed, and stretched points near the top of the lane to the top of the image, effectively squashing the lane ahead into a flat, overhead view. An example image is below.

![Transformed to overhead view][perspective transformed]

The image is not a true overhead perspective, as it is compressed significantly in y in order to hold as much signal as possible. Several more example images are shown in the notebook.


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
