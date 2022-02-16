# Project 1

## Project Overview
This project primarily dealt with the development of an SVG rasterization pipeline for points, lines, and triangles which can, optionally, be transformed, colored according to a barycentric system, or textured using an image. Special care was given throughout the project to generating high quality rasterization by supporting both super-sampling to improve quality for smoother edges and coloring where continuously defined as well as more interesting texture oriented anti-aliasing features like bi-linear pixel sampling and mipmap level interpolation. When taking all of these features together, we've created a not only fairly capable but indeed high quality SVG rendering engine which can be used to display very complex images.

Through this project, I feel that I've learned a *lot* about different anti-aliasing techniques. Specifically, while I feel I had a general grasp on the continuous-mipmap level texture filtering technique after our course lectures on the topic, actually implementing it provided a much stronger understanding of it. Additionally (and somewhat unexpectedly), this project gave me a much deeper understanding of the LLVM compiler infrastructure as I attempted to root-cause what turned out to be a miscompilation stemming from incorrect vectorization. This tangent, while not exactly relevant to the class, was interesting both to my own interests in programming languages/compiler research as well as because it gave me a much more low-level look at triangle intersection code.

## Part 1 (Triangle rasterization)
My implementation uses a very simple sampling based method for rasterizing triangles.

First, we compute the bounding box for each triangle:

![](assets/task1/1.png)

This is done by simply computing the minimum and maximum x and y coordinates across all of the triangle's vertices. This bounding box gives us a much smaller sample space from which we can perform intersection tests and is thus performance critical.

Next, we arbitrarily define the three lines L\_{01}, L\_{12}, L\_{20} and consider only the portion of those lines that falls within the earlier defined bounding box (i.e. the line tangent vectors):

![](assets/task1/2.png)

Then, for each point in the bounding box, we can consider which "side" of the line it lies on. If the pixel lies on the same side of each line, we know the pixel must be contained within the triangle and may thus color that pixel appropriately. This leads to the following final rasterization result for `basic/test4.svg` (default viewing parameters):

![](assets/task1/task1_4svg.png)

While this works and the triangles are in fact rasterized correctly, this does lead to a great deal of aliasing. One particularly extreme case of this can be seen in the pixel inspector where the red triangle, in addition to its abundant jaggies, appears to become discontinuous for a period since it becomes so narrow as to not be considered inside of any of the pixels in the region. This is the correct behavior but it is not very visually appealing.

From a performance standpoint, we can conclude that this algorithm is no worse than one that checks every sample within the bounding because this algorithm is constrained to checking *only* the pixels in the bounding box. In other words, this algorithm is not worse than one that does this because the algorithm presented here does precisely that.

## Part 2 (Antialiasing by supersampling)
Conceptually, supersampling antialiasing (SSAA) works to combat aliasing by simulating a higher sampling frequency (thereby increasing the Nyquist frequency) and then lowering the higher frequency samples into the lower frequency screen sample rate. In effect, this means we render a scene at a much higher resolution and then downsampling (typically by averaging neighboring samples). This is useful in practice since it allows us to provide much smoother edge transitions.

To add SSAA to our rasterization pipeline, we had to make a handful of changes. 

To begin, we defined our SSAA rate as the number of pixels that are unified to create one super pixel. This means that an SSAA rate of 4x means four pixels (arranged in a 2x2 square) are averaged together to form each screen pixel. We store this square-rooted rate (ie the dimension of one edge of the square) in the rasterizer's `sample_rate` field as we have no actual use for the non-square rooted size and because computing the square root is computationally expensive and thus something worth caching.

Next, since we now wish to render more pixels, we need an appropriately large place to store them. This meant that we needed to scale up the `sample_buffer` by a factor of  `sample_rate` in each dimension. This is very expensive (costing 16x more memory for 16x SSAA) but doing so vastly simplifies the pipeline since it allows us to directly render the entire scene into the sample buffer and then, later, downsample into the frame buffer instead of attempting to perform both operations simultaneously.

With this new, expanded sample buffer, we now need to render into it. This required two changes. 

First, to draw points and lines correctly (and un-supersampled per the spec), we simply modified the `fill_pixel` to translate from frame buffer coordinates to sample buffer coordinates by scalding each dimension up by `sample_rate` and then drawing a cluster a `sample_rate` by `sample_rate` square centered approximately at that newly scaled up position. This preserves the existing behavior for points and lines since the later down sampling algorithm will merge all the pixels in this square into one super pixel in the frame buffer.

Finally, with all elements rendered into the frame buffer, we are now prepared to reduce the sample buffer into the frame buffer. We did this by modifying the `resolve_to_framebuffer` function to iterate the frame buffer and average the colors of the corresponding `sample_rate` by `sample_rate` square of pixels in the sample buffer into one super pixel in the frame buffer. Once this processing step is complete, the SSAA image is now available in the frame buffer in its final form.

As a result of all this, triangles have significantly smoother edges since the sharp transition which appears in the sample buffer is averaged away into an appropriately smooth edge which approximates the very high frequency change, as desired. This provides a very visually appealing effect:

| 1x SSAA | 4x SSAA | 16x SSAA |
|:---:|:---:|:---:|
|![](assets/task2/screenshot_2-13_21-38-8_4_ss_1.png)|![](assets/task2/screenshot_2-13_21-38-10_4_ss_4.png)|![](assets/task2/screenshot_2-13_21-38-13_4_ss_16.png)|

By revisiting the pointy end of red triangle we observed in part 1, we can very clearly observe both the effect and extent of the varying levels of SSAA. SSAA at 1x, as expected, results in no changes since the super pixels are made up of just one sample each. In 4x, however, we start to see significant improvement both in the jaggedness and the size of the discontinuity because SSAA 4x is able to communicate varying levels of "edge present" through *hue* (unlike its binary 1x counterpart) due to its supersampling behavior. Thus, SSAA 4x is able to smoothly communicate the edge fall off behavior happening inside a single pixel by observing the edge fall off behavior across the four sub-samples. In 16x, we see an effect similar to 4x but slightly more pronounced. Notably, this rendition has no visible discontinuities. This is possible in 16x but not 4x because the larger sample footprint of 16x allows it to capture the extremely skinny part of the triangle due to its much smaller sub-pixels and thus much higher sampling frequency.




















