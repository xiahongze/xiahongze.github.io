---
layout: post
title: Ray Tracing wih CUDA 001
date: 2023-12-31 12:00:00 +1100
categories: tech c++ cuda raytracing
comments: true
---

# Ray Tracing with CUDA (001)

My journey into the realm of parallel programming has been a thrilling one, fueled by years of fascination and a drive to harness the power of multi-core CPUs in my everyday work. This passion took root during my PhD research, where I delved into High Performance Computing (HPC) and experimented with MPI, stretching calculations across multi-node clusters. As a professional software developer and an avid PyTorch user, I've come to appreciate how PyTorch leverages GPU capabilities and efficiently utilizes all available computer cores. However, this convenience comes with a level of abstraction that leaves me yearning for a deeper understanding of the underlying mechanisms.

Seeking to quench this thirst for knowledge, I embarked on a learning adventure with Caltech's [CUDA course](http://courses.cms.caltech.edu/cs179/), which proved to be an eye-opener. This experience inspired me to explore further applications of this technology, leading me to the fascinating world of ray tracing - a concept I've been enamored with since my days playing video games. GPUs, with their innate knack for graphics rendering and parallel computing, are perfectly suited for this task.

In my quest for comprehensive understanding, I stumbled upon the legendary ["one weekend" series of posts](https://raytracing.github.io/) by Peter Shirley. To date, I have devoured the first two series, applying my newfound CUDA skills to implement the code for the initial series and the BVH tree for the sequel. Roger Allen's [CUDA implementation](https://github.com/rogerallen/raytracinginoneweekendincuda) served as my springboard into this exciting domain, and I highly recommend his enlightening post on the NVIDIA tech blog [here](https://developer.nvidia.com/blog/accelerated-ray-tracing-cuda/).

Now, I am poised to chronicle my experiences and insights in a series of posts, hoping to enlighten and inspire those who share an interest in this captivating topic.

## CPU -> GPU

Understanding the distinction between CPUs and GPUs in computing is like comparing a skilled multitasker to a team of specialized workers. CPUs, akin to a versatile multitasker, excel at handling a few complex tasks simultaneously. They boast a limited number of cores, often ranging from 4 to 12 in typical desktop computers. Each core is a powerhouse in its own right, capable of performing multiple operations simultaneously, a feat known as hyper-threading. CPUs shine in sequential tasks, thanks to their rich instruction sets and sophisticated branch prediction algorithms.

Conversely, GPUs resemble a large team of workers, each focused on a simple, specific task. With potentially thousands of cores, GPUs can handle numerous tasks at once, albeit each core is limited to one task at a time. For instance, the Intel i7-8700K CPU has 6 cores, each operating at 3.7 GHz. In contrast, the GeForce GTX 1080 Ti, a GPU from yesteryears, boasts 3584 cores, although at a lower 1.48 GHz each.

When it comes to executing a single task, CPUs often outpace GPUs. However, GPUs take the lead in scenarios requiring the simultaneous execution of many tasks, particularly in fields like matrix operations and graphics rendering. It's important to note that GPUs aren't universally faster than CPUs; their efficiency depends on the nature of the task and its implementation. Tasks that are predictable and computationally intensive are prime candidates for GPU acceleration.

## Ray Tracing

Ray tracing is one of such tasks, as each ray shooting from the camera can be calculated independently and there is going to be hundreds of thousands of rays for a typical scene. It is based on the idea that we can trace the path of light and simulate the way light interacts with the objects in the scene. The basic idea is that we shoot a ray from the camera and see if it hits any objects in the scene. If it does, we calculate the color of the object at the intersection point and then shoot another ray from the intersection point to see if it hits any other objects. We repeat this process until we hit the light source or we reach the maximum number of bounces. The color of the pixel is the color of the light source if we hit the light source or the background color if we don't hit anything. Check out `rasterization` and `ray tracing` [here](https://en.wikipedia.org/wiki/Rasterisation) and [here](<https://en.wikipedia.org/wiki/Ray_tracing_(graphics)>) for more details.

To compute the interaction between ray and objects, we can naively iterate through all objects in a scene and check if the ray hits any of them. This is called `brute force` method and it is very slow and that is the method implemented in the first series of Peter Shirley's posts. To speed up the process, we can use a data structure called `Bounding Volume Hierarchy (BVH)` tree. The basic idea is that we divide the scene into smaller and smaller boxes and we can quickly determine if a ray hits a box. If it does, we can then check the objects in the box to see if the ray hits any of them. If it doesn't, we can discard the box and move on to the next box. Check out [here](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy) for more details. You can think about this as a binary search algorithm built specifically for ray tracing. This is the method implemented in the second series of Peter Shirley's posts.

It makes a lot of sense to use BVH tree to eliminate unnecessary computations. However, trees are not very friendly to GPUs because of the memory access pattern. Recursion is something GPU does not quite like. Hence, direct translation of Peter's code into CUDA does not lead to efficient computation and I need to improve on that. Luckily, it turns out we can flatten the tree structure and make it more GPU friendly. This is called `linearization` of BVH tree and it is a crucial optimization technique. The way that this is done in my code is to first build the tree in the CPU and then flatten it into a linear array. This array is then copied to the GPU and used for ray tracing. It works very well and I am very happy with the performance. To give you some idea, here is progress so far:

My machine: AMD5800X + RTX3080 + 32GB RAM

Test scene: 1200 x 800 pixels, 100 samples per pixel, 50 bounces, random spheres (488 objects)

1. Pure CPU code (brute force) from Peter Shirley's book ~ around 4min40s on single core. _I actually did not sample 100 rays per pixel. Instead, I used 10 per pixel and
   it took around 28.2s on single core. Hence, I estimated that it would take around 282s for 100 samples per pixel._
2. Unoptimized CUDA code from Roger Allen's code ~ around 15s on GPU
3. Optimized CUDA code (brute force) ~ around 6s on GPU
4. Optimized CUDA code with BVH tree ~ around 0.5s on GPU

As can be seen and unsurprisingly, GPU is much faster than CPU and BVH tree is much faster than brute force method.

![Random Sphere with Defocus Effect](/assets/img/random-spheres.jpg)

## CUDA

There are a few learnings I would like to share about using CUDA.

1. Memory access patterns matter. Choose the right grid/block/thread size matters. But do not over optimize.
2. Prefer `cudaMalloc` over dynamic memory allocation in CUDA. It is much faster.
3. In my experience, there is no gain to parallelize samples over the pixels. Just parallelize over the rays.
4. Compile each `.cu` file into a `.o` file and then link them together. This results in much faster code than compiling everything in header files.
   I understand that Mr Roger Allen uses only header files because he wants to make it easier for readers to understand the code & logic but not the
   mechanics of C++. This might be due to the extensive use of inlining among other things. It definitely works better for me to use
   separate `.h` and `.cu` file with a proper `Makefile` to compile and link them all.

## Final Words

It has been a very interesting journey and I have learnt a lot. All of my code will be made public once I have finished implementing the second final scene from Peter Shirley's book. In the meantime, please feel free to leave a message if you have any questions or suggestions. I am more than happy to discuss with you.
