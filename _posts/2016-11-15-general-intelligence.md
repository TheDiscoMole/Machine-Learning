---
layout: post
title: "General Intelligence"
date: 2016-11-15
excerpt: "ongoing AGI research & learning project"
project: true
tag:
comments: false
---
This project is a playground for me to play around with and research the backbone functionality of neural networks. It started with a dislike for the rigid structure of neural networks compared to the brain's neurogensis and synaptogenesis capabilities, but developed into something of a passion project in varied regions of general intelligence subjects.

**1 Framework**

**1 [Halloc]()**: GPU hardware on both the Nvidia and AMD stack are not built for kernels to efficiently do parallel DRAM writes. In order to implement neurogensis and synaptogenesis into the gradient descent algorithm, with as little overhead as possible, each neuron and synapse need to have the capacity to free and allocate memory from the kernel to *grow* or *die*. This is a custom-fit, header-only re-implementation of the [Halloc](https://github.com/canonizer/halloc) GPU kernel memory management project.

**2 Extension**: Currently this project is shaping up to a fully stand alone [CUDA extension with PyBind11](), but significant quality of life could be gained from changing this to be a PyTorch extension. Though due to the vast fundamental differences of core underlaying compute and update functionality there may be a lot of inefficient patchwork needed to make both libs in sync.

**3 Framework**: The Framework will be a bare-bones reimplementation of the most basic functionality found in existing frameworks like PyTorch and Tensorflow with a few conceptual differences to facilitate dynamic graph structures. The main differences will be:

* **static layer size**: any layer with will be subdivided into a subset of small neuron groups of predefined size allowing the graph evolve to be as dense or sparse as possible.

* **module interface**: while the Framework retains 3 core functions for modules to compute the output, the gradients and apply the gradients. The 3rd function will need to evolve the the cell. It is currently up for debate whether a final 4th function will recompile changed subgraph topologies or the Framework will employ a run-as-you-go kernel. Both solutions would have pros and cons in terms of work and efficiency.

**4 Future**: There are several higher
