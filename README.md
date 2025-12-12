# Single Sample Per Pixel Convolution

Single Sample per Pixel (SSPP) is a stochastic trick to improve the performance of otherwise expensive convolution operations.

In classic convolution, a filter kernel is shifted across an image and at each location each kernel pixel is multiplied with each overlapping image pixel, and then summed together. For large filter kernels this calculation is expensive.

With SSPP instead the kernel is transformed into a cumulative probability density function and then at each shifting position sampled to select only a single pixel from the input image. This decreases the cost from being linear in kernel size to being constant time per input image pixel.

For sampling the probability distribution a precalulated source of randomness (of proper size) can be used.
