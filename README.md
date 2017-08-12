
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

[//]: # ![Uncalibrated chessboard][uncalibrated chessboard]
![Calibrated chessboard][chessboard calibrated]

The IPython notebook shows each calibration image and its undistorted pair.


### Pipeline (single images)

I applied a series of transformations to each image to find the lane lines.

#### 1. Undistortion
For each image, I removed camera distortion using the aforementioned `undistort` method. An example base image and undistorted image are below.

![Base image with distortion][base image]
![Image with distortion removed][undistorted]


#### 2. Filters!

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `another_file.py`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

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
