---
layout: post
title: Ray Tracing wih CUDA 001
date: 2023-12-31 12:00:00 +1100
categories: tech c++
comments: true
---

# Ray Tracing with CUDA (001)

I have been obsessed with parallel programming in years and have been constantly thinking about how to better make use of multi-core CPUs in my daily work. Before I became a professional software developer, I had been using HPC for my PhD research projects and had written some code with MPI and ran calculations across multi-node cluster. I am now also a pytorch user and understand that pytorch uses GPU when available and definitely can use all the cores in a computer. The thing here is that a lot of stuff has been abstracted away from normal users and for me, this is not particularly satisfying because I want to know how things work under the hood. Therefore, I took a [CUDA course](http://courses.cms.caltech.edu/cs179/) offered by Caltech and found it very interesting. I decided to do more with this technology and started to think about what else I can apply. Naturally, I thought about ray tracing. I have been a fan of ray tracing since ages ago when I played video games. GPUs are born to render graphics and parallel computing.

When you search for explanations about ray tracing, you can't miss the legendary [series posts](https://raytracing.github.io/) by Peter Shirley. As of today, I have read the first two series and implemented the code in CUDA for the first series and also the BVH tree for the second series. I have used Roger Allen's [code](https://github.com/rogerallen/raytracinginoneweekendincuda) as a starting point of my adventure and please take a look at his post at nvida tech blog [here](https://developer.nvidia.com/blog/accelerated-ray-tracing-cuda/). I am going to write a series of posts to document my journey and hopefully it will be useful for someone who is interested in this topic.

## CPU -> GPU

We know that CPU is good at sequential computing and GPU is good at parallel computing. What exactly does this statement mean? CPU is designed to execute a few very complex tasks at a time. It has a few cores and each core can execute a single task at a time. GPU is designed to execute a lot of simple tasks at a time. It has a lot of cores and each core can execute a single task at a time. Take Intel i7-8700K as an example, it has 6 cores and each at run at 3.7 GHz. If you look at the olden day GeForce GTX 1080 Ti, it has 3584 cores and each at run at 1.48 GHz. For single task execution, CPU is much faster than GPU. However, if we have a lot of tasks to execute, GPU can be much faster than CPU. This is especially true for tasks that can be parallelized, such as matrix operations and graphics rendering. This observation does not mean that GPU is always faster than CPU. It depends on the task and how it is implemented. Predictable and intensive tasks are good candidates for GPU acceleration.

## Ray Tracing

Ray tracing is one of such tasks, as each ray shooting from the camera can be calculated independently and there is going to be hundreds of thousands of rays for a typical scene. It is based on the idea that we can trace the path of light and simulate the way light interacts with the objects in the scene. The basic idea is that we shoot a ray from the camera and see if it hits any objects in the scene. If it does, we calculate the color of the object at the intersection point and then shoot another ray from the intersection point to see if it hits any other objects. We repeat this process until we hit the light source or we reach the maximum number of bounces. The color of the pixel is the color of the light source if we hit the light source or the background color if we don't hit anything. Check out `rasterization` and `ray tracing` [here](https://en.wikipedia.org/wiki/Rasterisation) and [here](<https://en.wikipedia.org/wiki/Ray_tracing_(graphics)>) for more details.

To compute the interaction between ray and objects, we can naively iterate through all objects in a scene and check if the ray hits any of them. This is called `brute force` method and it is very slow and that is the method implemented in the first series of Peter Shirley's posts. To speed up the process, we can use a data structure called `Bounding Volume Hierarchy (BVH)` tree. The basic idea is that we divide the scene into smaller and smaller boxes and we can quickly determine if a ray hits a box. If it does, we can then check the objects in the box to see if the ray hits any of them. If it doesn't, we can discard the box and move on to the next box. Check out [here](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy) for more details. You can think about this as a binary search algorithm built specifically for ray tracing. This is the method implemented in the second series of Peter Shirley's posts.

It makes a lot of sense to use BVH tree to eliminate unnecessary computations. However, trees are not very friendly to GPUs because of the memory access pattern. Recursion is something GPU does not quite like. Hence, direct translation of Peter's code into CUDA does not lead to efficient computation and I need to improve on that. Luckily, it turns out we can flatten the tree structure and make it more GPU friendly. This is called `linearization` of BVH tree and it is a crucial optimization technique. The way that this is done in my code is to first build the tree in the CPU and then flatten it into a linear array. This array is then copied to the GPU and used for ray tracing. It works very well and I am very happy with the performance. To give you some idea, here is progress so far:

My machine: AMD5800X + RTX3080 + 32GB RAM

Test scene: 1200 x 800 pixels, 100 samples per pixel, 50 bounces, random spheres (488 objects)

1. Pure CPU code from Peter Shirley's book ~ around 10 min or more on single core
2. Unoptimized CUDA code from Roger Allen's code ~ around 15s on GPU
3. Optimized CUDA code (brute force) ~ around 6s on GPU
4. Optimized CUDA code with BVH tree ~ around 0.5s on GPU

As can be seen and unsurprisingly, GPU is much faster than CPU and BVH tree is much faster than brute force method.

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
