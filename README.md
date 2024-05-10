
# FiltersOptimizationInfo
Community tests and information page on the proposed method for optimizing texture readings for two-dimensional filters using difference derivatives

# Idea
Proposed method uses fine difference derivatives in fragment shader to reduce the number of texture samples from *(2r+1)*<sup>*2*</sup> to *(r+1)*<sup>*2*</sup> in two-dimensional filters with local area radius *r*. In fragment shader you have `genType dFdxFine(genType p)` and `genType dFdyFine(genType p)` functions (GLSL is used). They calculate derivatives using local differencing based on the value of p for the current fragment and its immediate neighbor. We should define
```GLSL
bvec2 isOdd =  bvec2(ivec2(gl_FragCoord.xy) &  1);
vec2 t =  vec2(isOdd) *  -2.0  +  1.0;
```
to make a data swap between fragments in 2x2 group.
Example:
```GLSL
vec3 p = vec3(ivec2(gl_FragCoord.xy) & 1, 0.0);
// p values in 2x2 group
// [ vec3(0, 0, 0), vec3(1, 0, 0),
//   vec3(0, 1, 0), vec3(1, 1, 0) ]

// swap along the x-axis
p += dFdxFine(p) * t.x;
// p values in 2x2 group
// [ vec3(1, 0, 0), vec3(0, 0, 0),
//   vec3(1, 1, 0), vec3(0, 1, 0) ]

// swap along the y-axis
p += dFdyFine(p) * t.y;
// p values in 2x2 group
// [ vec3(1, 1, 0), vec3(0, 1, 0),
//   vec3(1, 0, 0), vec3(0, 0, 0) ]
```
# How you can use it?
The idea is similar to shared memory in compute shader --- single reading of unique texels and shared access to them. But here you need to exchange partial sums for filters rather than texels. Otherwise it will decrease performance --- *r(3r+2)* fine derivatives are very expensive!

# What filters are implemented?
 - Gaussian filter: <https://github.com/kzubatov/gaussian_filter>
	 - Default version is implemented in fragment shader and reads *(2r+1)<sup>2</sup>*
 texels
	 - Proposed version uses fine derivatives and reads *(r+1)<sup>2</sup>* texels
	 - Compute version is two-pass version implemented in compute shader using shared memory. 128x1x1 workgroups are used
	 - Linear version is implemented in fragment shader and using linear sampler. It reads *2r+2* texels. See original paper: <https://www.rastergrid.com/blog/2010/09/efficient-gaussian-blur-with-linear-sampling>
  - Bilateral filter: <https://github.com/kzubatov/bilateral_filter>
	- Default version is implemented in fragment shader and reads *(2r+1)<sup>2</sup>*
 texels
	- Proposed version uses fine derivatives and reads *(r+1)<sup>2</sup>* texels
	- Compute version is implemented in compute shader using shared memory. 16x16x1 workgroups are used
 - Tent filter: <https://github.com/kzubatov/tent_filter>
	 - Default version is implemented in fragment shader and reads *(2r+1)<sup>2</sup>*
 texels
	- Proposed version uses fine derivatives and reads *(r+1)<sup>2</sup>* texels
	- Compute version is implemented in compute shader using shared memory. 16x16x1 workgroups are used
	- Important moment:
		Classic kernel for tent filter is *g(i, j) = max(k - b * abs(i), 0) * max(k - b * abs(j), 0)*. We use modified kernel to *g(i, j) = k - b * (abs(i) + abs(j))*. k and b are variable parameters in both cases. It was done in order to investigate performance of filters with kernel rank 2. For classic kernel use the same proposed method as for Gaussian filter
 - Variance clipping, TAA: <https://github.com/kzubatov/taa_statistics>
	 - Default version is implemented in fragment shader and reads (2r+1)<sup>2</sup>
 texels
	- Proposed version uses fine derivatives and reads (r+1)<sup>2</sup> texels
	- Compute version is implemented in compute shader using shared memory. 16x16x1 workgroups are used
	- Important moment:
	Our shader calculates mean and standard deviation for variance clipping, because they require sampling in local area. No other parts of TAA are implemented

# Detailed explanation of the proposed method
The paper is under review. Link will be up as soon as possible. For now, try experimenting with the source code)

# Tests. It would be great to see your tests here! (via pull requests)

This fork of Vulkan sample collection <https://github.com/kzubatov/shaders_benchmark> was used to launch shaders on RTX 2070 and Adreno 650. Performance was measured using Vulkan timestamp queries. You can use it too or write your own engine and rewrite shaders. Please, use the comments section to describe your changes to the shaders (or C++ code) if you will be changing them. It would be great to desribe how you measure performance (nsight, radeon profiler, timestapms, renderdoc etc).

---
Author: <https://github.com/kzubatov>
GPU: RTX 2070
Resolution: 1920x1080
Comments: Performance was measured using Vulkan timestap queries. Shaders from repos above.
| Gaussian filter, time in ms | Default | Proposed | Linear | Compute |
|-----------------------------|:-------:|:--------:|:------:|:-------:|
| 3x3 | 0,402 | 0,305 |	0,529 |	0,865 |
| 5x5 | 0,762 | 0,406 |	0,538 |	0,805 |
| 7x7 | 1,264 | 0,593 |	0,512 |	0,914 |

<br>

| Bilateral filter, time in ms | Default | Proposed | Compute |
|------------------------------|:-------:|:--------:|:-------:|
| 3x3 | 0,405 | 0,304 | 0,465|
| 5x5 | 0,762 | 0,512 | 0,729|
| 7x7 | 1,262 | 1,069 | 1,055|

<br>

| Tent filter, time in ms | Default | Proposed | Compute |
|-------------------------|:-------:|:--------:|:-------:|
| 3x3 | 0,403 | 0,298 | 0,440 |
| 5x5 | 0,802 | 0,402 | 0,765 |
| 7x7 | 1,291 | 0,557 | 1,139 |

<br>

| Variance clipping, time in ms | Default | Proposed | Compute |
|-------------------------|:-------:|:--------:|:-------:|
| 3x3 | 0,404 | 0,311 | 0,444 |
| 5x5 | 0,769 | 0,448 | 0,756 |
| 7x7 | 1,250 | 0,787 | 1,320 |

---

Author: <https://github.com/kzubatov>
GPU: Adreno 650
Resolution: 2400x1080
Comments: Performance was measured using Vulkan timestap queries. Shaders from repos above.
| Gaussian filter, time in ms | Default | Proposed | Linear | Compute |
|-----------------------------|:-------:|:--------:|:------:|:-------:|
| 3x3 | 0,983 |	1,543 | 0,616 | 5,343 |
| 5x5 | 3,436 |	2,210 | 0,685 | 6,172 |
| 7x7 | 6,548 |	4,528 | 0,887 | 6,234 |

<br>

| Bilateral filter, time in ms | Default | Proposed | Compute |
|------------------------------|:-------:|:--------:|:-------:|
| 3x3 |	2,363 |	4,070 |	3,697 |
| 5x5 |	6,576 |	4,387 |	6,528 |
| 7x7 |	9,073 |	6,984 |	10,370 |

<br>

| Tent filter, time in ms | Default | Proposed | Compute |
|-------------------------|:-------:|:--------:|:-------:|
| 3x3 |	0,980 |	1,534 | 3,425 |
| 5x5 |	2,855 |	2,425 | 5,798 |
| 7x7 |	6,347 |	6,226 | 9,907 |

<br>

| Variance clipping, time in ms | Default | Proposed | Compute |
|-------------------------|:-------:|:--------:|:-------:|
| 3x3 |	2,175 |	2,033 |	4,278 |
| 5x5 |	5,788 |	3,519 |	6,087 |
| 7x7 |	8,056 |	6,495 |	9,540 |
---

