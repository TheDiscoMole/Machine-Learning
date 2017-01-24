---
layout: post
title:  "Simple Neural Nets"
date:   2017-01-07
excerpt: "A simple Neural Network implementation for complex graph stuctures in Python, C++ and CUDA"
tag:
- Machine Learning
- contest
- Codingame
comments: false
---

Most Simple Neural Network implementations on github (or wherever else) suffer either from convoluted explanations or a rigid layered configurations. Has the brain, the inspiration for Neural Nets, proven to be so precisely layered? No, networks are complex in structure and that is something a NN implementation should embrace. We will be looking at a very simple Python implementation of a complexly layered graph. If you just care about the code, here are the links:

* [Python](https://github.com/TheDiscoMole/Simple-Complex-Neural-Net/tree/master/Python)
* [C++](https://github.com/TheDiscoMole/Simple-Complex-Neural-Net/tree/master/CPP)
* [CUDA](https://github.com/TheDiscoMole/Simple-Complex-Neural-Net/tree/master/CUDA)

------------------------------------------------------------------

**layer.py** will contain the 3 types of layers in our [feed-forward](https://en.wikipedia.org/wiki/Feedforward_neural_network) graph structure. Input, Hidden and Output. When constructed, these require:

* names which will be used to identify the layers for things like connecting them with Synapses
* sizes which define how many neurons are present in each layer
* activation functions to apply to the neural inputs
* loss functions in the case of the output layer which will help propogate gradients against target data

lets first do the simple stuff, activation & loss functions and their respective derivatives:

{% highlight py %}
# sigmoidal activation function & derivative
class Sigmoid:
    
    __call__ = lambda self,x: 1 / (1 + np.exp(-x))
    
    def gradient(self,x,g):
        s = self(x)
        return s * (1 - s) * g

# linear activation function & derivative
class Linear:
    
    __call__  = lambda self,x: x
    gradient  = lambda self,x,g: g
{% endhighlight %}

{% highlight py %}
# binary cross entropy and derivative
class CrossEntropy:

    __call__ = lambda self,y,t: - t * np.log(y) - (1 - t) * np.log(1 - y)
    gradient = lambda self,y,t: (t - y) / ((y - 1) * y)

# mean squared error and derivative
class MSE:
    
    __call__ = lambda self,y,t: ((y - t) / 2) ** 2
    gradient = lambda self,y,t: 2 * (y - t)
{% endhighlight %}
