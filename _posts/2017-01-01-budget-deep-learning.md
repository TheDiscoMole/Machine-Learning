---
layout: post
title:  "Budget Deep Learning"
date:   2016-03-15
excerpt: "A reasonably functional example of a budget Deep Learning computer build."
tag:
- markdown 
- syntax
- sample
- test
- jekyll
comments: false
---

A few months ago I build a budget Deep Learning only machine and had trouble finding many good recources online, so here is a list for anyone who is dealing with a similar situation. The machine costs less than $700 with 6GB of device memory and one of the fastest CUDA chips of the current generation. If you are looking to understand the hardware constraints more percisely, such as the recommended PCIe capabilities I was unaware of, *[this tutorial](http://timdettmers.com/2015/03/09/deep-learning-hardware-guide/)* is an amazing place to start. But if you are just looking for a well functioning build to roughly copy, here it is: (all links to products are through partpicker to ensure compatibility)

| Part | Product | Price |
|:--------|:-------:|--------:|
| CPU   | Intel Core i3-6100   | ~110   |
| GPU   | Zotac GeForce GTX 1060 6GB 6GB Mini   | ~250   |
| RAM   | Kingston HyperX Fury Black 16GB (2 x 8GB) DDR4-2133   | ~110   |
| SSD   | Kingston SSDNow V300 Series 120GB 2.5"   | ~45   |
| Moth   | ASRock H110M-HDS Micro ATX LGA1151   | ~45   |
| Case   | Rosewill FBM-02 MicroATX Mini   | ~25   |
| Power   | EVGA 500W 80+ Bronze   | ~50   |
|=====
| Total Price  |   | ~630
{: rules="groups"}

**[List on Partpicker](https://pcpartpicker.com/list/hvX33F)** (not sure if this does expire someday)

**NOTE** 

* use a Linux distro as your OS (like Arch-Linux) cause it makes the use deep learning tools, libraries and drivers so much easier
* unless you are planning to overclock (which is pointless for deep learning, you dont need anything other than factory coolers)
* a better CPU and Motherboard might make sense if you are able to spare the coin and plan to upgrade over time with dual GPU.
* thanks to the low power consumption craze, power supplies can be bought at even lower Watt-age and price.
