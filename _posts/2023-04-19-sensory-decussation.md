---
layout: post
title: "The Brain as an Autoencoder: Sensory Decussation and the Corpus Callosum"
date: 2023-04-19
excerpt: "An exploration of autoencoder sub-graph implementations"
tag:
comments: false
---

For several year I have worked under a simple assumption, that the brain is, at least partially, an autoencoder. Over those years papers exploring this concept for Neural Network pretraining have shown some interesting results. The earliest example I could find was:

[Split-Brain Autoencoders: Unsupervised Learning by Cross-Channel Prediction](https://arxiv.org/abs/1611.09842)

There are other papers that such as this that explain Network implementation and performance results, in regards to pretraining, in detail. In this post I wish to go into the structural and intuitive reasons why this exact thing is likely happening in our brains in some permanent form, and why we might wish to consider it our neural networks in general.

------------------------------------------------------------------

## Intuition

Modelling the brain as a constant autoencoder explains a lot of our unsupervised learning behaviours. Learning to move faster/more efficiently through a new apartment at night without ever having to stub your toe, hearing your name called out in a loud room without your mommy giving you a hug every time, etc. The brain clearly has some sort of "unsupervised" approach to encoding our sensory inputs which doesn't require direct feedback. It makes sense of our internal representation of reality to always slowly approach some balanced, encoded state and I think this is the predominant use of our corpus callosum. My personal hypothesis is that our sensory inputs are effectively split into 2 separate masks, during the sensory decussation process, before being fed into their respective hemispheres. This would mean each hemisphere is only seeing about half the information, but because the each sensory input consists of so much data each hemisphere still gets more than enough information to infer the whole.

To simplify the argument lets only consider visual information for now, but note that this obviously has no problem extending to all senses. Consider the following diagram:

<img style="float: right;" width="300" src="https://upload.wikimedia.org/wikipedia/commons/thumb/4/4d/Neural_pathway_diagram.svg/800px-Neural_pathway_diagram.svg.png">

Non-arbitrarily simplified, according to my interpretation, each eye would be sending a checkerboard mask of it's visual input to each hemisphere.
