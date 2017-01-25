---
layout: post
title:  "Fantastic Bits"
date:   2016-11-26
excerpt: "My failed late attempt at an online AI contest"
tag:
- Machine Learning
- contest
- Codingame
comments: false
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/xhgEOLJlbjw"> </iframe>

**[Fantastic Bits](https://www.codingame.com/leaderboards/challenge/fantastic-bits/country/de)** was an AI competition on Codingame where the goal was write a game AI which was going to compete in a 2D version of Quidditch. The entire contest lasted 10 days, but I only found out 3 days before the deadline. The plan was to write a game state simulator for heuristic tree search. (like how chess AIs work) Though I had just started getting to know C++ in depth, thanks to [this useful blog post](http://files.magusgeek.com/csb/csb_en.html) from someone who had competed in a similar previous competition I almost had my simulation finished and was going to train a Neural Network to learn the tree search heuristic using the [NEAT](https://en.wikipedia.org/wiki/Neuroevolution_of_augmenting_topologies) algorithm by making the species compete against each other. Had I finished the simulation with only an hour to spare the heuristic would have probably been a simple score evaluation, so essentially a random [Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo_method) search copout.

The code containing my simulation of the game can be found **[here](https://github.com/TheDiscoMole/Fantastic-Bits-Simulation)**. 

This code is full of bugs due to rushed coding and understanding of game mechanics, but can be used a sim template for future challenges. {: .notice}
