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

**layer.py** will contain the 3 types of layers in our [feed-forward](https://en.wikipedia.org/wiki/Feedforward_neural_network) graph structure. `Input`, `Hidden` and `Output`. When constructed, these require:

* names which will be used to identify the layers for things like connecting them with Synapses
* sizes which define how many neurons are present in each layer
* activation functions to apply to the neural inputs
* loss functions in the case of the output layer which will help propogate gradients against target data

lets first do the simple stuff, `activation` & `loss` functions and their respective `derivatives`:

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

Now we are ready to construct our layers lets make lay down some design choices. An individual neural needs to contain 3 float values for its state:

* bias
* pre-activation
* post-activation

Additionally we require 2 bits of functionality from each layer:

* feed-forward propogration; consists of summing the synaptic outputs from the input neurons together with the layer bias and computing the activation function. 
* gradient descent back propogation; sums the synaptic input gradients from the output neurons, computes the gradient over the activation function and updates the bias.

If you wish to learn the intricacies of gradient descent, this is a good place to start even though there is a confusing mistake in the last part.

<iframe width="560" height="315" src="https://www.youtube.com/embed/5u0jaA3qAGk" frameborder="0" allowfullscreen></iframe>

{% highlight py %}

# hidden layer
class Hidden:
    
    def __init__(self, name, size, activation):
        self.name = name
        self.activation = activation
        
        # input & output vertexes and numpy state matrix
        self.xs = 0
        self.ns = np.zeros((3,size), dtype=float)
        self.ys = 0
        
        # used to check if layer has been computed in this
        # complex graph structure
        self.computed = False
    
    # overload __call__ operator for feed forward function
    def __call__(self, state):
        # check if the layer has been computed in this state
        if self.computed != state:
            ns = self.ns
            
            # sum inputs with bias
            ns[1] = ns[0] + sum(x(state) for x in self.XS())
            # compute elementwise activation
            ns[2] = self.activation(ns[1])
            
            self.computed = state
        
        # return layer outputs
        return np.copy(self.ns[2])
    
    # gradient descent
    def gradient(self, state):
        # check if the layer has been computed in this state
        if self.computed != state:
            ns = self.ns
            
            # sum output gradients
            ns[2] = sum(y.gradient(state) for y in self.YS())
            # compute elementwise activation gradient
            ns[1] = self.activation.gradient(ns[1],ns[2])
            # update bias
            ns[0]-= .1 * ns[1]
            
            self.computed = state
        
        # return gradients
        return np.copy(self.ns[1])
    
    # generator which iterates all input verteces
    def XS(self):
        current = self.xs
        while current:
            yield current
            current = current.x_next
    
    # generator which iterates all output verteces
    def YS(self):
        current = self.ys
        while current:
            yield current
            current = current.y_next
{% endhighlight %}
