---
layout: post
title: Ray Tracing wih CUDA 001
date: 2023-12-31 12:00:00 +1100
categories: tech c++
comments: true
---

# Ray Tracing with CUDA (001)

I have been obsessed with parallel programming in years and have been constantly thinking about how to better make use of multi-core CPUs in my daily work. Before I became a professional software developer, I had been using HPC for my PhD research projects and had written some code with MPI and ran calculations across multi-node cluster. I am now also a pytorch user and understand that pytorch uses GPU when available and definitely can use all the cores in a computer. The thing here is that a lot of stuff has been abstracted away from normal users and for me, this is not particularly satisfying because I want to know how things work under the hood. Therefore, I took a [CUDA course](http://courses.cms.caltech.edu/cs179/) offered by Caltech and found it very interesting. I decided to do more with this technology and started to think about what else I can apply. Naturally, I thought about ray tracing. I have been a fan of ray tracing since ages ago when I played video games. GPUs are born to render graphics and parallel computing.

When you search for explanations about ray tracing, you can't miss the legendary [series posts](https://raytracing.github.io/) by Peter Shirley. As of today, I have read the first two series and implemented the code in CUDA for the first series and also the BVH tree for the second series. I have used Roger Allen's [code](https://github.com/rogerallen/raytracinginoneweekendincuda) as a starting point of my adventure and please take a look at his post at nvida tech blog [here](https://developer.nvidia.com/blog/accelerated-ray-tracing-cuda/). I am going to write a series of posts to document my journey and hopefully it will be useful for someone who is interested in this topic. For general context, I also attach a `concepts` section at the end of this post.

## CPU -> GPU

We know that CPU is good at sequential computing and GPU is good at parallel computing. What exactly does this statement mean? CPU is designed to execute a few very complex tasks at a time. It has a few cores and each core can execute a single task at a time. GPU is designed to execute a lot of simple tasks at a time. It has a lot of cores and each core can execute a single task at a time. Take Intel i7-8700K as an example, it has 6 cores and each at run at 3.7 GHz. If you look at the olden day GeForce GTX 1080 Ti, it has 3584 cores and each at run at 1.48 GHz. For single task execution, CPU is much faster than GPU. However, if we have a lot of tasks to execute, GPU can be much faster than CPU. This is especially true for tasks that can be parallelized, such as matrix operations and graphics rendering.

## Concepts

### What is ray tracing?

Ray tracing is a technique for rendering 3D graphics with high quality. It is based on the idea that we can trace the path of light and simulate the way light interacts with the objects in the scene. The basic idea is that we shoot a ray from the camera and see if it hits any objects in the scene. If it does, we calculate the color of the object at the intersection point and then shoot another ray from the intersection point to see if it hits any other objects. We repeat this process until we hit the light source or we reach the maximum number of bounces. The color of the pixel is the color of the light source if we hit the light source or the background color if we don't hit anything. Check out `rasterization` and `ray tracing` [here](https://en.wikipedia.org/wiki/Rasterisation) and [here](<https://en.wikipedia.org/wiki/Ray_tracing_(graphics)>) for more details.

### What is CUDA?

CUDA is a parallel computing platform and programming model developed by NVIDIA for general computing on GPUs. It allows software developers and software engineers to use a CUDA-enabled graphics processing unit (GPU) for general purpose processing – an approach termed GPGPU (General-Purpose computing on Graphics Processing Units). The CUDA platform is a software layer that gives direct access to the GPU's virtual instruction set and parallel computational elements, for the execution of compute kernels. [Wikipedia](https://en.wikipedia.org/wiki/CUDA)

### What is BVH tree?

BVH stands for Bounding Volume Hierarchy. It is a tree structure that allows us to quickly find the intersection between a ray and the objects in the scene. The basic idea is that we divide the scene into smaller and smaller boxes and we can quickly determine if a ray hits a box. If it does, we can then check the objects in the box to see if the ray hits any of them. If it doesn't, we can discard the box and move on to the next box. Check out [here](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy) for more details.

#### Linearization of BVH tree

The linearization of BVH for GPU computation is a crucial optimization technique. GPUs, being massively parallel processors, prefer data structures with regular, predictable access patterns. Linearizing a BVH involves flattening the tree structure into a linear array that can be efficiently traversed by the GPU. This transformation typically involves ordering the nodes of the tree in a way that reduces memory jumps during traversal, which is critical for maintaining high performance in GPU-based computations. This linearization enables faster traversal speeds, making it highly suitable for real-time applications like gaming and interactive simulations, where rapid rendering is essential.
