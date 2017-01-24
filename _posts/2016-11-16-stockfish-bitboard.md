---
layout: post
title:  "Stockfish Bitboard"
date:   2016-11-16
excerpt: "The dissected code of the Stockfish bitboard and move generator for those who wish to test they chess AI skills."
tag:
- Machine Learning
- Chess
- Stockfish
comments: false
---

To test the performance of my Neurogensis & Synaptogenesis work I, much like every other AI nerd, wanted to see how well it would perform playing a game. Since chess is such a cliche and well established AI community, why not see how an evolving Neural Net performes against existing powerfull engines. None of the major engines use any form of Deep Learning and should probably be classified as Monte Carlo Expert Machines instead of AIs. While dissecting the Stockfish bitboard and move generator from source I realised there would be probably be others out there looking for a legal chess move generator such as this to test their AI. 

**TL;DR** 

Just want to use Stockfish to generate all legal moves from a position? **[look no further](https://github.com/TheDiscoMole/Stockfish-BitBoard)**
