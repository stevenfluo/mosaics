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

#  CS 180: Project 4, Steven Luo

# Overview

Meow



# Part 1. Defining Correspondences

I started by finding several pairs of images, cropping them so they were the same dimensions and the subjects were lined up (similar size and aspect ratio), then manually labelling correspondence points between each image in each pair. In order to include the full image in subsequence morphs, corners are added as correspondence points. I had anywhere from 45 to 100+ correspondence points! I found that for faces with glasses, the additional points were neccessary to prevent distortion in the glasses frames and ensure a smooth animation. Thanks to a past student for their [helpful tool](https://cal-cs180.github.io/fa23/hw/proj3/tool.html) in labeling my images.

To illuminate the intermediate steps, I turn to Barack Obama and George Clooney:

<p align="center">
    <img src="./img/obama.jpg" alt="obama" width="33%"/>
    <img src="./img/george.jpg" alt="george" width="33%"/>
    <p style="text-align: center;"><i>Barack Obama and George Clooney.</i></p>
</p>

Here's what they looked like after defining correspondences and calculating a Delaunay triangulation from these points:

<p align="center">
    <img src="./img/obamamesh.jpg" alt="obama" width="33%"/>
    <img src="./img/georgemesh.jpg" alt="george" width="33%"/>
    <p style="text-align: center;"><i>Barack Obama and George Clooney.</i></p>
</p>

For future cleaning-up efforts, I would make it so that I add the corners directly in the code instead of trying to click on the corners — this would fix the thin edges around some of my results.

# Part 2. Computing the "Mid-way Face"

To compute the "mid-way" face between images, I first compute the average shape by taking the mean of keypoint locations, then warp each face to that shape, and finally average the colors together. 

I find the average shape by taking the mean across coordinates (i.e. (1, 2) and (3, 4) would have a mean of (2, 3)). 

To implement the warp function, I compute the Delaunay triangulation from the averaged set of correspondence points. Note that by indexing into the two original sets of points using the mid-way image simplices, we are able to maintain correspondences between triangles. To find the matrix defining the affine transformation from a triangle from the original image to a triangle from the target image given the original and target points, I create a "design" matrix, multiply it with a reshaped vector of target points, and selecting the appropriate values to turn it into a well-formed affine transformation matrix. I found [this comment](https://edstem.org/us/courses/64731/discussion/5354090?comment=12536116) on Ed particularly helpful. Using `skimage.draw.polygon`, I can select all the points within a triangle, apply the affine transformation matrix, and interpolate values from the original to the target triangle.

It turns out that forward mapping will actually produce a lower-quality morph, because there isn't necessarily a one-to-one correspondence to the transformed image! To fix this, we use inverse mapping. For each triangle, we apply the inverse of the affine transformation matrix to points from the transformed image given by `skimage.draw.polygon`. Since the corresponding point on the original image is not necessarily integer-valued, we interpolate using `scipy.interpolate.griddata` with nearest neighbors to fill in the pixel value on the target triangle from the original image. This successfully produces a warped image!

To create the midway shape, we want to blend the two warped original images together. This works because each of the original images has been warped to the same shape, so by cross-dissolving the images we get a good "morph" of the input images. My `cross-dissolve` function is simply a weighted average of the two image arrays, controlled by the weight parameter such that smaller weights emphasize the first image and larger weights emphasize the second. For these results, I use an unweighted mean such that both images are equally emphasized.

<p align="center">
    <img src="./img/obama.jpg" alt="obama" width="30%"/>
    <img src="./img/george.jpg" alt="george" width="30%"/>
    <img src="./img/task2.jpg" alt="morph" width="30%"/>
    <p style="text-align: center;"><i>Barack Obama, George Clooney, and the mid-way face.</i></p>
</p>

<p align="center">
    <img src="./img/task2obamawarp.jpg" alt="obama" width="30%"/>
    <img src="./img/task2georgewarp.jpg" alt="george" width="30%"/>
    <p style="text-align: center;"><i>Barack Obama and George Clooney morphed to the mid-way shape.</i></p>
</p>


# Part 3. The Morph Sequence

I implemented the `morph` function to create an animation that seamlessly transitions from one image to another. This integrates my `warp_image` and `cross_dissolve` functions, essentially repeating part 2 but with different "mid-way" images. 

<p align="center">
    <img src="./img/obamageorge_final_reverse.gif" alt="obama morph" width="50%"/>
    <p style="text-align: center;"><i>Barack Obama <=> George Clooney</i></p>
</p>

<p align="center">
    <img src="./img/steven_clara_reversed.gif" alt="obama morph" width="50%"/>
    <p style="text-align: center;"><i>Steven (me) <=> Clara (my friend)</i></p>
</p>

<p align="center">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/DqcI0fttMBM" 
          frameborder="0" allow="accelerometer; autoplay; clipboard-write; 
          encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</p>


# Part 4. The "Mean Face" of a Population

I used the spatially normalized frontal face images and the corresponding annotated shapes from the [FEI Face Database](https://fei.edu.br/~cet/facedatabase.html) to find the average face of a subpopulation of 97 images — smiling men. Here are some examples:

<p align="center">
    <img src="./img/1b.jpg" alt="obama" width="30%"/>
    <img src="./img/30b.jpg" alt="george" width="30%"/>
    <img src="./img/82b.jpg" alt="morph" width="30%"/>
    <p style="text-align: center;"><i>Smiling men from the FEI Face Database.</i></p>
</p>

I loaded the corresponding annotated shapes then manually added the corners of the each image as keypoints as well. First, I computed the average face shape of this subpopulation, then morphed each of the faces in the dataset to the average shape. 

<p align="center">
    <img src="./img/task4_1b.jpg" alt="obama" width="30%"/>
    <img src="./img/task4_30b.jpg" alt="george" width="30%"/>
    <img src="./img/task4_82b.jpg" alt="morph" width="30%"/>
    <p style="text-align: center;"><i>Smiling men from the FEI Face Database morphed to the average shape</i></p>
</p>

I also warped my face into the average geometry of the subpopulation, and the average face warped into my geometry. The results were disorienting (and probably would have been better if my image wasn't at an angle or if I wasn't wearing glasses -- you can see how the result is a bit distorted on the sides) but interesting to look at.

<p align="center">
    <img src="./img/task4_mean_img.jpg" alt="obama" width="30%"/>
    <img src="./img/caricature_alpha_0.0.jpg" alt="obama" width="30%"/>
    <p style="text-align: center;"><i>Subpopulation mean and me.</i></p>
</p>

<br />

<p align="center">
    <img src="./img/task4me_mesh.jpg" alt="obama" width="40%"/>
    <img src="./img/task4avg_mesh.jpg" alt="obama" width="40%"/>
    <p style="text-align: center;"><i>Face meshes on input images.</i></p>
</p>

<br />

<p align="center">
    <img src="./img/task4_metoavg.jpg" alt="george" width="30%"/>
    <img src="./img/task4_avgtome.jpg" alt="morph" width="30%"/>
    <p style="text-align: center;"><i>Me warped to the average geometry, and the average face morphed to my geometry.</i></p>
</p>

# Part 5. Caricatures: Extrapolating from the Mean

`alpha` controls how emphasized my features or the average face's features are. Lower alphas emphasize the average face, and vice versa.

<div class="grid-container">
        <div class="grid-item">
            <img src="./img/caricature_alpha_-1.0.jpg" alt="caricature_alpha_-1.0">
            <p>alpha = -1.0</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_-0.75.jpg" alt="caricature_alpha_-0.75">
            <p>alpha = -0.75</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_-0.5.jpg" alt="caricature_alpha_-0.5">
            <p>alpha = -0.5</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_-0.25.jpg" alt="caricature_alpha_-0.25">
            <p>alpha = -0.25</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_0.0.jpg" alt="caricature_alpha_0.0">
            <p>alpha = 0.0</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_0.25.jpg" alt="caricature_alpha_0.25">
            <p>alpha = 0.25</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_0.5.jpg" alt="caricature_alpha_0.5">
            <p>alpha = 0.5</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_0.75.jpg" alt="caricature_alpha_0.75">
            <p>alpha = 0.75</p>
        </div>
        <div class="grid-item">
            <img src="./img/caricature_alpha_1.0.jpg" alt="caricature_alpha_1.0">
            <p>alpha = 1.0</p>
        </div>
</div>

In the future, I could improve my results with better image cropping/alignment/posing.

# Bells and Whistles

## Changing Ethnicity

With my friend Andrew's permission, I changed the ethnicity of a photo of his face using the [average English man's face](https://pmsol3.wordpress.com/2011/04/07/world-of-averages-europeave/). 

<p align="center">
    <img src="./img/andrew.jpg" alt="obama" width="30%"/>
    <img src="./img/whiteguy.jpg" alt="george" width="30%"/>
    <p style="text-align: center;"><i>Andrew and the average English man's faces.</i></p>
</p>

<p align="center">
    <img src="./img/andrew_whiteguy_shapeonly.jpg" alt="obama" width="30%"/>
    <img src="./img/andrew_whiteguy_both.jpg" alt="george" width="30%"/>
    <img src="./img/andrew_whiteguy_appearanceonly.jpg" alt="morph" width="30%"/>
    <p style="text-align: center;"><i>Shape-only, both, and appearance-only morphs of Andrew and the average English man.</i></p>
</p>

<p align="center">
    <img src="./img/andrew_whiteguy_shapeonly.gif" alt="obama" width="30%"/>
    <img src="./img/andrew_whiteguy_both.gif" alt="george" width="30%"/>
    <img src="./img/andrew_whiteguy_appearanceonly.gif" alt="morph" width="30%"/>
    <p style="text-align: center;"><i>Animated transitions of shape-only, both, and appearance-only morphs.</i></p>
</p>

## Morphing Video on a Theme

I dug up more than a dozen old yearbook photos of myself, and made a movie about how my face has changed over time. I set it to Ride of the Valkyries by Richard Wagner, because I'm watching Wagner's opera Tristan und Isolde after I finish my midterms. I've been a huge fan of classical music even since I was a kid (my go-to radio station was always [KDFC](https://www.kdfc.com)), and my current earworm is Glinka's [Overture to Ruslan and Ludmila](https://www.youtube.com/watch?v=QW3IwWFkpfc).

Check it out here!

<p align="center">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/pvG2WS9P7uE" 
          frameborder="0" allow="accelerometer; autoplay; clipboard-write; 
          encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</p>

## Face-Morphing Music Video of Students in the Class

To create this music video, I worked with my fellow SEED Scholars taking this class to source images and turn it into a morphing sequence! The program took professional portraits of everyone two years ago, which made alignment and labelling significantly easier, resulting in nice and smooth transitions. It features music that we felt describes the vibes of this project.

This video features David, Steven, Clara, and Camille, in that order.

<p align="center">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/OgBwQ1yH06E" 
          frameborder="0" allow="accelerometer; autoplay; clipboard-write; 
          encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</p>
