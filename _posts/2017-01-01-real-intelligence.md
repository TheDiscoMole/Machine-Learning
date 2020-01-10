---
layout: post
title:  "Improving Neural Networks"
date:   2017-01-01
excerpt: "My silly attempt at Strong AI."
project: true
tag:
- Artificial Intelligence 
- Neurogenesis
- Synaptogenesis
- Recursion
- Parallelism
comments: false
---

There is still much to be done about the core functionality behind Neural Networks and there are several impactful differences between artificial and biological. This is a long standing project and interest that comes in waves and I work on it on and off depending inspiration and desire.

* **Neurogenesis** & **Synaptogenesis**: Probably the largest glaring hole in networks these days compared to the brain is their rigid sctructure and inability to truly change. I have developed a mechanism that allows an individual net to grow and prune neurons and synapses in approximataly the right places. This algorithm is highly parallelisable and integrates "seemlessly" with gradient descent. This is currently in development stages as it requires a complete rewriting of core pyTorch-like functionality in CUDA with a custom GPU memory allocator.

* **Unsupervised Learning**: If we look at human capability to optimize behaviour without much incentive it becomes obvious that some form of unsupervised learning exists. (by supervised implying things such as pain and clearly emotion/hormone/etc driven incentives to change behaviour) These mechanisms don't quite explain the brains tendency to optimize smaller things very well. For example our ability to gradually become more able to walk through our own home in the dark. I think this passive learning stems largely from 2 hemispheres and [corpus callosum
](https://en.wikipedia.org/wiki/Corpus_callosum) which might cause the hemispheres to have, at least partially, an autoencoder functionality. 

* **Fractalisation**: Our brains are inherently highly self-similar and while this is also true for Neural Networks currently, it isn't sufficiently so. For me an ideal graph would not just grow and prune neurons and synapses, but entire graphs and sub-graps, given the network the ability to grow in and outwards by treating subgraphs as undividual nodes and vice versa. This is highly theorical and I haven't even botherd complicating my first bulletpoint by trying to implement it, but it is on the list.
