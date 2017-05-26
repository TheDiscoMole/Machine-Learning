---
layout: post
title:  "Improving Neural Networks"
date:   2017-01-01
excerpt: "My silly attempts at real AI."
project: true
tag:
- Artificial Intelligence 
- Neurogenesis
- Synaptogenesis
- Recursion
- Parallelism
comments: false
---

I am working on what I perceive Neural Networks are supposed to look like. Here is a little run down:

* **Neurogenesis** & **Synaptogenesis**: Finishing Stages. My Neural Networks can grow, shrink, rewire themselves and update synaptic weights in one *relatively* clean parallel algorithm. Currently I am working on a scalable CUDA implementation that can learn from real data-streams.

* **Unsupervised Learning**: Prototyping stages. Training data will be completely optional as parts of the Neural Net will be acle to learn purely from input data. This currently works, but is not really at implementation stage for scalable code yet. It requires restructuring the current design paradigm for my afore-mentioned Neural Net quite a bit. 

* **Fractalisation**: Concept stages. Using inheritance, the network objects will ideally become completely self similar and allow for growing, pruning and rewiring of entirely interconnected sub-graphs. Thus, higher level learning.

* **Distributability** Concept stages. By implementing several communication interfaces for neurons/graphs (eg. DMA & ZMQ), a network should be able to be hosted over different configurations of hardware at all times.

*This is a long standing project as i repeatedly go on months long hiatuses and work on it as my interest comes and goes.*
