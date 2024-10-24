---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: Home
permalink: /
---

<style>
        .grid-container {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 20px;
            text-align: center;
        }
        .grid-item {
            display: flex;
            flex-direction: column;
            align-items: center;
        }
        .grid-item img {
            width: 100%;
            height: auto;
        }
        .grid-item p {
            font-style: italic;
        }
</style>

<!-- MathJax -->
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

# **Part A: Image Warping and Mosaicing**

# Overview

In this project, I used two or more photographs to create image mosaics by registering, projective warping, resampling, and compositing them. I computed homographies to warp images, and used a modified version of the Laplacian stack to blend images into the final mosaic.

# Part A.1: Taking Photos and Recovering Homographies

To capture photos that blend well, I took them in a way such that the transforms between them are projective (a.k.a. perspective) by fixing the center of projection and rotating my camera while capturing photos.

A homography is a [projective transformation](https://www.sciencedirect.com/topics/engineering/homography) between two planes (a mapping between two planar projections of an image). In other words, homographies are simple image transformations that describe the relative motion between two images, when the camera (or the observed object) moves.

We can represent the transformation with a 3x3 matrix H, such that 

$$H \cdot p_{\text{original}} = p_{\text{transformed}}$$

$$\begin{bmatrix} h_1 & h_2 & h_3 \\ h_4 & h_5 & h_6 \\ h_7 & h_8 & 1 \end{bmatrix} \begin{bmatrix} x \\ y \\ 1 \end{bmatrix} = \begin{bmatrix} wx' \\ wy' \\ w \end{bmatrix}$$

We can solve for H by solving a system of equations with eight degrees of freedom, using the keypoints we generated using [this tool](https://cal-cs180.github.io/fa23/hw/proj3/tool.html). Extending this to our full set of points gives us:

$$\begin{bmatrix}
x_1 & y_1 & 1 & 0 & 0 & 0 & -x'_1x_1 & -x'_1y_1 \\
0 & 0 & 0 & x_1 & y_1 & 1 & -y'_1x_1 & -y'_1y_1 \\
x_2 & y_2 & 1 & 0 & 0 & 0 & -x'_2x_2 & -x'_2y_2 \\
0 & 0 & 0 & x_2 & y_2 & 1 & -y'_2x_2 & -y'_2y_2 \\
\vdots & \vdots & \vdots & \vdots & \vdots & \vdots & \vdots & \vdots \\
x_n & y_n & 1 & 0 & 0 & 0 & -x'_nx_n & -x'_ny_n \\
0 & 0 & 0 & x_n & y_n & 1 & -y'_nx_n & -y'_ny_n \\
\end{bmatrix}
\begin{bmatrix}
h_1 \\
h_2 \\
h_3 \\
h_4 \\
h_5 \\
h_6 \\
h_7 \\
h_8 \\
\end{bmatrix}
=
\begin{bmatrix}
x'_1 \\
y'_1 \\
x'_2 \\
y'_2 \\
\vdots \\
x'_n \\
y'_n \\
\end{bmatrix}$$

We solve this with a least-squares to find the minimum norm solution, since with more than four points the system becomes overdetermined. While we can solve for an exact solution with just four points, we want more in order to make our homography recovery more robust to noise. Note that after appyling the homography matrix H, in order to recover $$x'$$ and $$y'$$ we need to divide by $$w$$.

# Part A.2: Image Warping and Rectification

I implemented a warping function that takes the source image and the homography matrix $$\mathbf{H}$$ corresponding to the transformation from the source to the target image. The warp function applies $$\mathbf{H}$$ to the corners of the source image, calculates dimensions of the output image, then draws a polygon mask using the transformed corners. For each point in the polygon, I reverse interpolate from the source image by applying $$\mathbf{H}^{-1}$$ to recover the corresponding point in the source image (this is vectorized by constructing matrices).

To test the function, I rectified several images. Rectification is performed using a homography to make a known rectangular object rectangular again, even if it does not visually appear rectangular. For each image, I selected four points defining the corners of a rectangular shape, calculated the homography to a set of coordinates forming a rectangle, and then warped the image. 

For source images, I drew on photos I took during my trip to New York City this summer. The first example is a vintage subway advertisement in the New York Transit Museum. 

<p align="center">
    <img src="./img/ad.jpg" alt="ad" width="30%"/>
    <img src="./img/ad_pts.jpg" alt="ad" width="30%"/>
    <img src="./img/ad_rectified.jpg" alt="ad" width="30%"/>
    <p style="text-align: center;"><i>Original, with points labeled, and rectified images.</i></p>
</p>

This museum is one of my favorites of all time, and is defintely worth the stop in Brooklyn for.

The second image comes from the ceiling of the [Rose Main Reading Room](https://www.newyorker.com/culture/cultural-comment/nypl-rose-reading-room-and-the-real-meaning-of-luxury-in-new-york-city) in the Stephen A. Schwarzman Building —— the New York Public Library's flagship location.

<p align="center">
    <img src="./img/nypl_full.jpg" alt="ad" width="30%"/>
    <img src="./img/nypl_pts.jpg" alt="ad" width="30%"/>
    <img src="./img/nypl_rectified.jpg" alt="ad" width="30%"/>
    <p style="text-align: center;"><i>Original, with points labeled, and rectified images.</i></p>
</p>

I spent an afternoon working here before going on a tour of the United Nations!

# Part A.3: Blending and Mosaicing

I implemented a blending function to combine warped images with the base image their homography matrix was calculated relative to. First, I calculated the size of the output image. Then, I placed each image on its correct position on a blank output image using information about how much the image shifted during the warping process — if I were to stack these two images, the overlapping features should line up (I confirmed this during implementation). Finally, I blend these two images together using weights calculated from distance transforms for the images to combine the low frequencies. I used a Laplacian stack-based or a distance transform tiebreaker-based blending method to combine the high frequencies (further elaboration below).

My first mosaic is of the third floor in Berkeley Way West. For this image, I warped the image of the left side of the field of view and blended it with the center view. Until I get a research position in BAIR, I'll be underneath the 8th floor :\)
<p align="center">
    <img src="./img/bwwleft.jpg" alt="ad" width="30%"/>
    <img src="./img/bwwlm_warped_im.jpg" alt="ad" width="30%"/>
    <img src="./img/bwwmid.jpg" alt="ad" width="30%"/>
    <p style="text-align: center;"><i>Original left BWW, warped left BWW, and original center BWW.</i></p>
</p>

<p align="center">
    <img src="./img/bww_left.jpg" alt="ad" width="50%"/>
    <p style="text-align: center;"><i>Resulting mosaic.</i></p>
</p> 


I dug up an old Canon [Powershot](https://www.dpreview.com/articles/6913931643/the-vertical-elph-remembering-canons-powershot-tx1-hybrid-camera) [TX1](https://global.canon/en/c-museum/product/dcc541.html) last week, and have been putting it to use! I took photos of a particularly busy corner of my room using my digicam, and blended the warped lower perspective onto the higher perspective.

<p align="center">
    <img src="./img/room1.jpg" alt="ad" width="30%"/>
    <img src="./img/room1_warped_im_new.jpg" alt="ad" width="30%"/>
    <img src="./img/room2.jpg" alt="ad" width="30%"/>
    <p style="text-align: center;"><i>Original lower room, warped lower room, and original upper room.</i></p>
</p>

<p align="center">
    <img src="./img/room12_new_highsigma.jpg" alt="ad" width="50%"/>
    <p style="text-align: center;"><i>Resulting mosaic.</i></p>
</p>


Here's the same scene, but shot on my iPhone.
<p align="center">
    <img src="./img/r1.jpg" alt="ad" width="30%"/>
    <img src="./img/r1_warped_im_new.jpg" alt="ad" width="30%"/>
    <img src="./img/r2.jpg" alt="ad" width="30%"/>
    <p style="text-align: center;"><i>Original lower room, warped lower room, and original upper room.</i></p>
</p>

<p align="center">
    <img src="./img/r12_new_highsigma.jpg" alt="ad" width="50%"/>
    <p style="text-align: center;"><i>Resulting mosaic.</i></p>
</p>

Personally, I prefer the "vibes" of the digicam photos —— what do you think?

My last mosaics come from Busan, South Korea. Gamcheon Culture Village is a beautiful site with a great view of the ocean, though it was swelteringly hot when I was there earlier this year. 

Here's the left side of the Gamcheon lookout warped and blended with the middle view:
<p align="center">
    <img src="./img/left.jpg" alt="ad" width="30%"/>
    <img src="./img/lm_warped_im_new.jpg" alt="ad" width="30%"/>
    <img src="./img/middle.jpg" alt="ad" width="30%"/>
    <p style="text-align: center;"><i>Original left Gamcheon, warped left Gamcheon, and original middle Gamcheon.</i></p>
</p>

<p align="center">
    <img src="./img/gamcheon_left_new.jpg" alt="ad" width="50%"/>
    <p style="text-align: center;"><i>Resulting mosaic.</i></p>
</p>

Here's the right side of the Gamcheon lookout warped and blended with the middle view:
<p align="center">
    <img src="./img/right.jpg" alt="ad" width="30%"/>
    <img src="./img/rm_warped_im_new.jpg" alt="ad" width="30%"/>
    <img src="./img/middle.jpg" alt="ad" width="30%"/>
    <p style="text-align: center;"><i>Original right Gamcheon, warped right Gamcheon, and original middle Gamcheon.</i></p>
</p>

<p align="center">
    <img src="./img/gamcheon_right_new.jpg" alt="ad" width="50%"/>
    <p style="text-align: center;"><i>Resulting mosaic.</i></p>
</p>

With several warped images relative to a fixed center image, I also implemented a function to automatically put together multi-image mosaics by tracking cumulative offsets and using the shifts from each warped image. In my blending function, I use distance transforms to create weights. I use a Gaussian filter to isolate the low frequencies of the images, which I combine with a weighted average. For the higher frequencies (the methodology to isolate these is the same as creating the Laplacian stack in [project 2](https://stevenfluo.github.io/filters-frequencies)). To create the final output image, I combined the low and high frequencies — just like collapsing the Laplacian stack.

I tried two approaches to blending the high frequencies: first, a weighted average just like for the low-frequencies:

<p align="center">
    <img src="./img/gamcheon_blended_laplace.jpg" alt="ad" width="90%"/>
    <p style="text-align: center;"><i>Resulting mosaic.</i></p>
</p>

Notice that because of the fine details in the image and slight misalignment between the two, you can spot blurry lines such as on the mural in the bottom middle of the photo.

My second approach used the distance transforms as a tiebreaker to select whether to use the warped image or the base image high frequencies at a particular point: 

<p align="center">
    <img src="./img/gamcheon_total.jpg" alt="ad" width="90%"/>
    <p style="text-align: center;"><i>Resulting mosaic.</i></p>
</p>

Notice that "scaly" artifacts appear (look at the mural again!), but these are also hard to notice unless you look very closely. 

Looking at these mosaics makes me feel like I'm back on vacation!
