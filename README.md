
## Advanced Lane Finding Project

The [IPython/jupyter notebook](https://github.com/gardenermike/finding-lane-lines-reprise/blob/master/lane_lines.ipynb) in this repo implements a heuristic-based approach to finding lane lines in images or video.
The basic algorithm is:
* Apply camera distortion correction to the raw images
* Find regions (i.e. lines) of strong gradient in X (using a sobel filter)
* Finding regions (again, hopefully lane lines) of the image with strong saturation
* Combining the gradient and saturation-filtered images and masking away regions unlikely to hold lane lines
* Perspective transforming the image to an overhead perspective to make parallel line finding easier
* Applying a series of windows to the resulting image to find the centers of greatest pixel intensity
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
My math is empirical: I started with the procedure in mind and just tweaked the numbers until they worked.

After all of the steps in the pipeline thus far were applied, I obtained simple images with a strong signal, as shown below.

![Preprocessed image ready for lane detection][preprocessed]


#### 4. Finding lines

With the images fully preprocessed, I move on to the lane detection. This is a two step process. The first step is in the `find_window_centroids` function.
My algorithm is to split the image into nine horizontal slices and find the center on the left and right of maximum signal in each slice. The details are a little more involved :).
I start by finding the maximum on left and right in the bottom quarter of the image, with the center of the image excluded to avoid noise. Starting with the resulting centers, I use a much narrower window in progressive slices up the image. If no signal is found, I instead assume a linear progression from the prior center. I also take the mean of the left and right lanes and then adjust the centroids to be equidistant to the original points, ensuring that the lines stay parallel.
I tinkered with this function quite a bit to get something that would work well over a variety of signals. The result works quite well. One image is below, but I have a set of them in the notebook to check out.

![Centroids][centroids]

Once I found the centers of mass of each of the windows, I just use numpy's `polyfit` (see the `find_curve` function) to fit a second-degree polynomial through the centers. With the resulting curves, I draw two lines and fill the space between them on an overlay image, then map the overlay onto the original and perspective shift back to the original perspective. Success!

![Overlay][overlay]


#### 5. Stats and smoothing

With my lanes found and plotted, I'm done for static images, but with video I can do better. Using an (approximate) pixels-per-meter measurement, I can calculate the distance to the lane line on either side (`get_distances_from_center`) and the radius of each lane line (`get_curve_radius`). While these stats are useful on their own and are written in each frame of video, they also allow me to perform a sanity-check on the data and to smooth my stats between frames, significantly reducing jitter and allowing me to drop problem frames entirely.

---

### Video

The final result on a fairly clean video turned out really good, I think.

I've got a [link here](./video-output/project_video_output.mp4) but you can also make it easy and just see it on [YouTube](https://youtu.be/WvvEoZHugVk)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
