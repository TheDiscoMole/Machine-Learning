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

Just want to use Stockfish to generate all legal moves from a position? **[look no further](https://github.com/TheDiscoMole/Stockfish-BitBoard)**

-------------------------------------------------------------------------

To test the performance of my Neurogensis & Synaptogenesis work I, much like every other AI nerd, wanted to see how well it would perform playing a game. Since chess is such a cliche and well established AI community, why not see how an evolving Neural Net performes against powerful existing engines. None of the major engines use any form of Deep Learning and should probably be classified as Monte Carlo Expert Machines instead of AIs. While dissecting the Stockfish bitboard and move generator from source I realised there would probably be others out there looking for a legal chess move generators such as this to test their AI. 

-------------------------------------------------------------------------

this is the front facing API, which acts only as a template to understanding how to write your own usefull interface.

{% highlight cpp %}
// Chess Tree Node
struct Node
{
    Node *parent, *children;
    
    Position position;
    float evaluation;
    
    Node(Position position, Move m) : position(position, m) {}
    Node() : position(Position()) {}
    
    // Set/Get postion FEN
    void   fen(const string& fen);
    string fen();
    
    // Check if draw, Only when no children generated
    // Repition gets checked here since stateinfo from source was deleted
    // ~children and check = checkmate
    // ~children and ~check = stalemate
    bool isDraw();
    
    // Turn info
    Color whoseTurn();
    int   whatTurn();
    
    // Gets position as convenient NN input format arr[8][8][12]
    float *getBits();
    
    // Generate and play all LEGAL moves from this position
    void playMoves();
};
{% endhighlight %}
