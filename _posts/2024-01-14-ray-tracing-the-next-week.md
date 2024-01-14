---
layout: post
title: Ray Tracing the Next Week
date: 2024-01-14 12:00:00 +1100
categories: tech c++ cuda raytracing
comments: true
---

# Ray Tracing the Next Week

It has been two weeks since I completed the first series of Peter Shirley's ray tracing books (see my previous post [here](https://xiahongze.github.io/tech/c++/cuda/raytracing/2023/12/31/ray-tracing-with-cuda-101.html)). I've since embarked on the second series and completed the full featured final scene, which I'll share in this post. I also fulfilled my promise to release my code to the public, which you can find [here](https://github.com/xiahongze/RayTracingManyWeeksWithCuda). I welcome any questions or suggestions, and I hope this code will be useful to those interested in this fascinating topic.

## Learnings from this exercise

There are a few notable learnings that I want to share in this post. The first thing is that debugging CUDA code that has virtual function is really painful. I spent a lot of time trying to figure out why my code was crashing while debugging and finally figured out that it was because I was calling a virtual function from a device function. I had tried many ways to enable debugging of device code, but none of them worked until I removed the virtual function in the base class. This is not particularly appealing to me because Peter Shirley's implementation relies heavily on virtual functions. Although I could refactor the code to avoid virtual functions, I decided to live with the design for now as it is easier for me to establish a baseline implementation and then refactor it later.

The second thing I learnt is that when and why copy/move contructors and assignment operators are called. It is not always obvious to me and took me quite a while to figure out. The reason I looked into this is because I wanted to avoid double pointers in my code. The use of double pointer (`**ptr`) can be confusing and for my use case, I just need an array of instances and thought I did not need a double pointer. However, I was wrong. Although I did manage to use a single pointer but it was not without a lot of pain. This is because objects are created uniquely ---- I need to use `new` per object and the fact I `new` an object, I already created one in the current heap. So when I assign this new object back to the position where the single pointer points to, copy assignment or move assignment is called. I did not gain much by using a pre-allocated space in an array. In this case, using a double pointer actually makes sense, because after I `new` an object and only need to pass the pointer to this new object to this double pointer (array of pointers), and no copy assignment or move assignment is called. I hope this makes sense. A simple example below hopefully illustrates this point.

```cpp
A* pre_allocated = A[10];

pre_allocated[0] = A(); // move assignment is called

A example = A();

pre_allocated[1] = example; // copy assignment is called

A** double_ptr = (A**) malloc(sizeof(A*) * 10);

double_ptr[0] = new A(); // no assignment is called, only store the pointer

double_ptr[1] = &example; // no assignment is called, only store the pointer
```

The third thing I learnt is that it is of extremeley importance to initialize CUDA `curandState` before using it. I was getting weird looking images because I did not initialise the random number generator properly. Things like missing an entire plane in the box model, weirdly uniform looking pattern for a scene with supposedly random objects, are quite likely due to this. It is also important to initialise the random number generator with different seeds for each thread. I used the following code to initialise the random number generator.

```cpp
__global__ void init_rand(curandState *state, int seed) {
    int idx = threadIdx.x + blockIdx.x * blockDim.x;
    curand_init(seed, idx, 0, &state[idx]);
}
```

It is important to point out that even when using a single thread, one should still use `curand_init` to init the state first.

## Closing thoughts

When rendering the final scene of week 2, my RTX3080 was definitely cooked to 100%, which is good to see since I stopped ethereum mining. It took about 78s to run through 1,000 rays per pixel for a 800x800 image, which is amazingly faster than using CPU. The speedup here is likely 100x. The resultant image is shown below and I stil think that it is a bit noisy due to the low light condition. In the next series, I think there are some interesting techniques that touch on this topic. I will be looking forward to it.

![final scene of week 2](/assets/img/out-final-scene-1000-glossy.jpg)
_final scene oif week 2_

There is a correction I need to make to my previous post. Careful readers probably have spotted that the glass sphere in the final scene of week 1 is not rendered correctly. It was pure reflection instead of refraction. I have made a mistake in the formula and have already corrected it in the code. Below is a corrected image with checkered floor.

![final scene of week 1](/assets/img/out-spheres-100-checker.jpg)
_final scene of week 1_

Happy ray tracing!
