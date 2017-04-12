##Writeup Template
###You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./output_images/test_undist.jpg "Undistorted"
[image2]: ./test_images/test5.jpg "Road Transformed"
[image3]: ./output_images/threshold_binary.png "Binary Example"
[image4]: ./output_images/warped.jpg "Warp Example"
[image5]: ./output_images/poly.png "Fit Visual"
[image6]: ./output_images/drawarea.png "Output"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code implementing this step is contained in the code cell titled by "collect coefficient for undistort" in the IPython notebook "CarND-Advanced-Lane-Lines.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]
####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of color and gradient thresholds to generate a binary image (The code for this step is contained in the code cells of the IPython notebook titled by "combine binary" located in "CarND-Advanced-Lane-Lines.ipynb").  Here's an example of my output for this step.

![alt text][image3]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for perspective transform is in the code cell titled by "get the matrix for transforming perspective" in the IPython notebook.  I chose the hardcode the source and destination points in the following manner:

```
src_bottom_left = [208,718]
src_bottom_right = [1092, 718]
src_top_left = [603, 447]
src_top_right = [678, 447]

# Destination points are chosen such that straight lanes appear more or less parallel in the transformed image.
sq_bottom_left = [320,720]
sq_bottom_right = [920, 720]
sq_top_left = [320, 1]
sq_top_right = [920, 1]
```
This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 603, 447      | 320, 1        |
| 208, 718      | 320, 720      |
| 1092, 718     | 960, 720      |
| 678, 447      | 960, 1        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

日本語で失礼します。
車線を検出するにはまず、視点を変更した画像から左右の車線のピクセルを収集し、それらを用いて二次多項式の係数を求めます。
矩形の枠を車線に辿らせて、その内部にある、値を持つピクセルを集めていきます。
まず、矩形のスタート地点を設定します。画像を左右それぞれ半分に分けた中で、ヒストグラムを作成しもっとも値が高い位置を左右それぞれの車線の開始位置とします。
あとは、矩形の中の値を持つピクセルを配列に追加していき、矩形を Y 軸方向にずらしながら、繰り返しピクセルを集めます。
最上部まで達したら、それらのピクセル座標を用いて二次多項式の係数を求めます。
また、一度、多項式の係数を求めたあとは、次のフレームの画像では、その多項式から得られる曲線の近傍のピクセルを集め、それを用いて多項式の係数を求めるようにします。これにより、処理の負荷を軽減します。

上記を実装したコードは CarND-Advanced-Lane-Lines.ipynb の中の
「get line pixel by sliding window」と 「get line pixel by polynomial」の見出しをつけたコードセルにあります。

コードを実行して得られた曲線を以下に掲載します。

![alt text][image5]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

日本語で失礼します。
ある点での曲線の曲率は多項式の係数より求められます。
画面の最下点での曲率を保持しておきます。
また、多項式を求める際に、ピクセルを現実での長さに変換した上で得られた係数を用いて曲率を計算します。
上記を実装したコードは CarND-Advanced-Lane-Lines.ipynb の中の
「get line pixel by sliding window」と 「get line pixel by polynomial」の見出しをつけたコードセルの下部で以下のようにして求めております。
```
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension
left_fit_cr = np.polyfit(lefty*ym_per_pix, leftx*xm_per_pix, 2)
right_fit_cr = np.polyfit(righty*ym_per_pix, rightx*xm_per_pix, 2)

# Calculate the new radii of curvature
y_left_eval = np.max(lefty)
y_right_eval = np.max(righty)
left_line.radius_of_curvature = ((1 + (2*left_fit_cr[0]*y_left_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
right_line.radius_of_curvature = ((1 + (2*right_fit_cr[0]*y_right_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])
```

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The code implementing this step is contained in the code cell titled by "define handler of each frame" in the IPython notebook "CarND-Advanced-Lane-Lines.ipynb".  

![alt text][image6]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

日本語で失礼します。
光が強い箇所では度々、誤検出が発生しました。おそらく、彩度が強すぎるため、黄色い線の外側を車線と間違えて検出してしまうものと思われます。
