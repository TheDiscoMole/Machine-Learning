---
layout: post
title:  "Simple Neural Nets"
date:   2017-01-07
excerpt: "A simple Neural Network implementation for complex graph structures in Python"
tag:
- Machine Learning
- Neural Networks
comments: false
---

Most Simple Neural Network implementations on github (or wherever else) suffer either from convoluted explanations or a rigidly layered configuration. Has the brain, the inspiration for Neural Nets, proven to be so precisely layered? No, networks are complex in structure and that is something NN implementations should embrace. We will be looking at a very simple Python code of a complexly layered graph.

------------------------------------------------------------------

`layer.py` will contain the 3 types of layers in our [feed-forward](https://en.wikipedia.org/wiki/Feedforward_neural_network) graph structure. `Input`, `Hidden` and `Output`. When constructed, these require:

* `names` which will be used to identify the layers for things like connecting them with Synapses
* `sizes` which define how many neurons are present in each layer
* `activation functions` to apply to the neurons
* `loss functions` in the case of the output layer, which will help propogate gradients against target data

lets first do the simple stuff, `activation` & `loss functions` and their respective `derivatives`:

{% highlight py %}
# sigmoidal activation function & derivative
class Sigmoid:
    
    __call__ = lambda self,x: 1 / (1 + np.exp(-x))
    
    def gradient(self,x):
        s = self(x)
        return s * (1 - s)

# linear activation function & derivative
class Linear:
    
    __call__  = lambda self,x: x
    gradient  = lambda self,x: 1
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

------------------------------------------------------------------

Now that we are ready to construct our layers lets make some design choices. An individual neuron needs to contain 3 `float` values for its state:

* bias
* pre-activation
* post-activation

Additionally we require 2 bits of functionality from each layer:

* `feed-forward propogration` consists of summing the synaptic inputs together with the layer bias and computing the elementwise activation function. 
* `gradient descent back propogation` sums the synaptic output gradients, computes the gradient of the elementwise activation function and updates the bias through Gradient Descent.

Since we are layering our graph in a complex manner it will occur that in every pass any layer may be called multiple times. So that we don't recalculate layers redundantly and break our Gradient Descent algorithm by applying gradients more than once, an easy fix will be used through a graph wide and layer local `boolean` state value used to check whether said layer has been `computed` before in the current pass.

------------------------------------------------------------------

If you wish to learn the intricacies of Gradient Descent, this is a good place, to start even though there is a small confusing mistake in the last video of the playlist on Gradient Descent.

<center>
    <iframe 
        width="560" 
        height="315" 
        src="https://www.youtube.com/embed/5u0jaA3qAGk" 
        frameborder="0" 
        allowfullscreen>
    </iframe>
</center>

------------------------------------------------------------------

{% highlight py %}
class Input:
    
    def __init__(self, name, size, activation):
        self.name = name
        self.activation = activation
        
        # init outputs and  state
        self.ns = np.zeros((3,size), dtype=float)
        self.ys = []
    
    # overload __call__ operator for feed forward function
    def __call__(self, xs, state):
        # check if the layer has been computed in this state
        if self.computed != state:
            ns = self.ns
            # load inputs and add bias
            ns[1] = xs + ns[0]
            # compute input activation
            ns[2] = activation(ns[1])
            # update state
            computed = state
        
        # return layer outputs
        return np.copy(self.ns[2])
    
    # input gradients
    def gradient(self, state):
        # check if the layer has been computed in this state
        if self.computed != state:
            ns = self.ns
            
            # sum output gradients
            ns[2] = sum(y.gradient(state) for y in self.ys)
            # compute elementwise activation gradient
            ns[1] = ns[2] * self.activation.gradient(ns[1])
            # update bias with learning rate = 0.1
            ns[0]-= .1 * ns[1]
            
            self.computed = state

class Hidden:
    
    def __init__(self, name, size, activation):
        self.name = name
        self.activation = activation
        
        # input & output vertexes and numpy state matrix
        self.xs = []
        self.ns = np.zeros((3,size), dtype=float)
        self.ys = []
        
        # used to check if layer has been computed in pass
        self.computed = False
    
    # overload __call__ operator for feed forward function
    def __call__(self, state):
        # check if the layer has been computed in this state
        if self.computed != state:
            ns = self.ns
            
            # sum inputs with bias
            ns[1] = ns[0] + sum(x(state) for x in self.xs)
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
            ns[2] = sum(y.gradient(state) for y in self.ys)
            # compute elementwise activation gradient
            ns[1] = ns[2] * self.activation.gradient(ns[1])
            # update bias with learning rate = 0.1
            ns[0]-= .1 * ns[1]
            
            self.computed = state
        
        # return gradients
        return np.copy(self.ns[1])

class Output:
    
    def __init__(self, name, size, activation, loss):
        self.name = name
        self.activation = activation
        self.loss = loss
        
        # init inputs and state
        self.xs = []
        self.ns = np.zeros((3,size), dtype=float)
    
    # overload __call__ operator for feed forward function
    def __call__(self, state):
        # check if the layer has been computed in this state
        if self.computed != state:
            ns = self.ns
            
            # sum inputs with bias
            ns[1] = ns[0] + sum(x(state) for x in self.xs)
            # compute elementwise activation
            ns[2] = self.activation(ns[1])
            
            self.computed = state
        
        # return layer outputs
        return np.copy(self.ns[2])
    
    # output gradient descent
    def gradient(self, ts, state):
        # compute errors
        errors = cost(ts)
    
        # check if the layer has been computed in this state
        if self.computed != state:
            ns = self.ns
            
            # loss gradients
            ns[2] = loss.gradient(ns[2], ts)
            # compute elementwise activation gradient
            ns[1] = ns[2] * self.activation.gradient(ns[1])
            # update bias with learning rate = 0.1
            ns[0]-= .1 * ns[1]
            
            self.computed = state
        
        # return errors
        return errors
    
    # output layer cost
    def cost(self, ts):
        return np.sum(self.loss(ns[2], ts))
{% endhighlight %}

------------------------------------------------------------------

Next we need a way to connect layers with synaptic links and each `Synapse` needs to have the same `forward` and `backward` functions. Thanks to the afore-linked playlist we know the relationship between input and output neurons is linear, which makes for some very neat gradients. We will need to deal with 2 gradient calculations, one to propogate gradients further down the net and the other to update the synaptic weights. The gradients wrt to the input become the output gradients multiplied by the weight matrix transposed, and the gradients wrt to the weights become the matrix multiplication of the input transposed and the output gradients:

{% highlight py %}
class Synapse:
    
    def __init__(self, x, y):
        # attach the input & output layer and construct the weight matrix
        self.x  = x
        self.ws = np.random.uniform(-1,1,(x.size,y.size))
        self.y  = y
        
        # attach synapse to input & output layer
        x.ys.append(self)
        y.xs.append(self)
    
    # overload __call__ for feed forward compute
    def __call__(self, state):
        return np.dot(self.x(state), self.ws)
    
    # compute synaptic gradient
    def gradient(self, state):
        # input gradients
        result  = np.dot(self.y.gradient(state), np.transpose(self.ws))
        # weight updates
        self.ws-= np.outer(self.x(not state), self.y.gradient(state))
        # return gradients
        return result
{% endhighlight %}

------------------------------------------------------------------

Finally we have all the tools we need to combine it all into a `Graph` class with a front facing interface for Neural Network training, where we would ideally have the following functionality:

* add `Input` layer
* add `Hidden` layer
* add `Output` layer
* add `Synapse` connection between 2 layers
* compute `Graph` output (feed-forward)
* `Gradient Descent` (backpropogate)

{% highlight py %}
# relate input string to relevant activation/loss function
Loss = {'entropy':CrossEntropy(), 'mse':MSE()}
Activation = {'sigmoid':Sigmoid() ,'linear':ReLu()}

class Graph:
    
    def __init__(self):
        self.xs = {} # input layers
        self.ns = {} # hidden layers
        self.ys = {} # output layers
        
        self.state = True
   
    def add_input(self, name, size, activation='linear'):
        self.xs[name] = Input(name, size, Activation[activation])
    
    def add_hidden(self, name, size, activation='sigmoid'):
        self.ns[name] = Node(name, size, Activation[activation]);
    
    def add_output(self, name, size, activation='sigmoid', loss='entropy'):
        self.ys[name] = Output(name, size, Activation[activation], Loss[loss])
    
    # connect layers with a synapse
    def connect(self, x, y):
        if x in self.xs: x = self.xs[x]
        if x in self.ns: x = self.ns[x]
        
        if y in self.ns: y = self.ns[y]
        if y in self.ys: y = self.ys[y]
        
        Synapse(x,y)
    
    # overload __call__ with feed forward computation
    def __call__(self, xs):
        # load each input layer with provided inputs
        for name in xs:
            self.xs[name](xs[name], self.state)
        # compute layer outputs
        result = {name:y(self.state) for name,y in self.ys.items()}
        # reset graph state and return results
        self.state = not self.state
        return result
    
    def gradient(self, xs, ts):
        # compute layer outputs
        ys = self(xs)
        # compute graph wide error & load output layers with cost gradients
        cost = sum(self.ys[n].gradient(ts[n], self.state) for n in ys)
        # gradient descent
        for x in self.xs.values():
            x.gradient(self.state)
        # reset graph state adn return cost
        self.state = not self.state
        return cost
{% endhighlight %}

**NOTE**: This code clearly hasn't been filled with error/consistency checks, so it only works, if the user knows how to use the interface correctly (eg. connecting layers in such a way they **DO NOT** cause closed loops). Additionally there is no trained model storage or load function, so if you wish to keep your trained progress, this would need to be implemented still.
