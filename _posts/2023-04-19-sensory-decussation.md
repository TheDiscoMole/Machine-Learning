---
layout: post
title: "Our Brain the Autoencoder: Sensory Decussation and the Corpus Callosum"
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

Modelling the brain as a constant autoencoder explains a lot of our unsupervised learning behaviours. Learning to move faster/more efficiently through a new apartment at night without ever having to stub our toe, hearing one's name called out in a loud room without mommy handing out hugs every time, etc. The brain clearly has some sort of approach to encoding our sensory inputs, which doesn't require direct feedback. It makes sense of our internal representation of reality to always slowly approach some balanced, encoded state and I think this is the predominant use of our corpus callosum. My personal hypothesis is that our sensory inputs are effectively split into 2 separate masks, during the sensory decussation process, before being fed into their respective hemispheres. This would mean each hemisphere is only seeing about half the information. But due to each sensory input consisting of so much data, each hemisphere still gets more than enough information to infer the whole.

To simplify the argument lets only consider visual information for now, but note that this obviously has no problem extending to all senses. Consider the following diagram:

<img align="right" width="300" src="https://upload.wikimedia.org/wikipedia/commons/thumb/4/4d/Neural_pathway_diagram.svg/800px-Neural_pathway_diagram.svg.png">

Non-arbitrarily simplified, according to my interpretation, each eye would be sending a checkerboard mask of it's visual input to each hemisphere. This information is then processed by the relevant synaptic centres and turned into a contextualized representation by each hemisphere independently. Finally these encodings are compared with each other across the corpus callosum to generate the brains equivalent of autoencoder gradients.

Autoencoder-like structure potentially being a necessity for unsupervised learning would be a decent explanation for the evolution of hemispheric processing to begin with.

In terms of Artificial Neural Networks it also makes intuitive sense as an improvement to supervised gradient descent. Gradient descent "suffers" from the issue of optimising into local minima. If we were to fully integrate decussation, with a sufficiently small learning rate, into the entire training process of our models the subgraph gradients could essentially act as minor nudges towards a final minima that encodes our data with less bias/noise.

------------------------------------------------------------------

## Implementation

The implementation is actually quite simple, it just requires a wrapper for encoder subgraphs, which perform the same task. One of these subgraphs represents our intended encoder for inference, the other(s) are only there during the training. We make this distinction because:

1. doubling the size of our network **in this way** isn't really worth any amount of improvement generally
2. each decussation essentially encodes the same thing, so we should only need 1.
3. by treating subgraphs as temporary we can keep them small, only to be used to improve the training process (they don't do inference, just some state encoding)

Given the nature of the decussation task, and taking inspiration from nature, it makes sense to structure our disposable subgraphs similarly to the encoder we will actually be using during inference. We can slim our encoder copy down to conserve memory requirements since it only serves autoencoder gradients. Here is a pseudo-code example:

```py
d_input = 512
d_encoder = 2048
d_decussation = 128

nhead = 8

encoderInput = torch.nn.Linear(d_input, d_encoder)
encoder = torch.nn.TransformerEncoderLayer(d_encoder, nhead)
encoderDecussation = torch.nn.Linear(d_encoder, d_decussation)

decussationInput = torch.nn.Linear(d_input, d_decussation)
decussation = torch.nn.TransformerEncoderLayer(d_decussation, nhead)
```

Our input features are the same for both models. During training we apply a mask and inverted mask to the input for the encoder and it's copy respectively.

```py
input = torch.randn((1,32,512))
mask = torch.randn((1,32,512)) > 0.5
```

To generate the relevant gradients we take the Mean Squared Error loss function across the mirrored encodings:

```py
encoder_input = encoderInput(input * mask.float())
encoder_output = encoder(encoder_input)

decussation_input = decussationInput(input * (~mask).float())
decussation_output = decussation(decussation_input)

criterion = torch.nn.MSELoss()
optimizer = torch.optim.SGD(model.parameters(), learning_rate, momentum=momentum, weight_decay=weight_decay)
loss = criterion(decussation_output, encoderDecussation(encoder_output))
```
