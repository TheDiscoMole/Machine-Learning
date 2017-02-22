---
layout: post
title:  "Twitch Renderer"
date:   2017-01-07
excerpt: "Renders Twitch Streams and Chat as videos frame by frame in numpy"
tag:
- Twitch
- Computer Vision
comments: false
---

I was interested in applying some Computer Vision techniques and play around with twitch streams. As Twitch chat is a big part of the experience, and especially the emotes are a very visual component, this code allows you to render streams on `low` quality at 30 fps and do the same for twitch chat. Due to latency concerns the irc data stream worker will download and keep an up to date database of Twitch emotes and badges (BTTV emotes too). Currently this database only downloads channel emotes when you watch said channel as emotes from other channels are used "relatively" rarely in streams compared to stream emotes and globals.

------------------------------------------------------------------

The API is really simple...

*render everythin*

{% highlight py %}
from twitchrender import Renderer

stream = Renderer('LIRIK')
for video, audio, chat in stream:
    # do stuff with frames
{% endhighlight %}

*render video and chat only*

{% highlight py %}
from twitchrender import Renderer

stream = Renderer('LIRIK', audio=False)
for video, chat in stream:
    # do stuff with frames
{% endhighlight %}

Each `Renderer` instance can fetch 1 stream frame by frame. All you have to do is initialise the `Renderer` object and the overloaded `__iter__` does the rest.

------------------------------------------------------------------

`Renderer.__init__(self, channel, video=True, audio=True, chat=True, flush=True)`

This is the renderer object used to mantain data stream workers.

**args:**

* *channel:* the channel whose stream you wich to render
* *video:* `bool` which decides whether to render the streams video frames (default=`True`)
* *audio:* `bool` which decides whether to render the streams audio frames (default=`True`)
* *chat:* `bool` which decides whether to render the streams chat frames (default=`True`)
* *flush:* this `bool` can be used to tell the renderer to only fetch to most recent frames so if your calls to `__iter__` generator happen less frequently than 30 times per second you an drop missed frames to ingest real time data. Otherwise the frame buffer will grow depending on that speed. (Default=`True`)

`Renderer.__iter__(self)`

This is the iterator used to `yield` data stream frames.

**returns**

* (360,640,3) video frame numpy array if `video` argument at `Renderer` intitialisation was set to `True`
* (44100/30,2) audio frame numpy array if `audio` argument at `Renderer` intitialisation was set to `True`
* (360,100,3) chat frame numpy array if `chat` argument at `Renderer` intitialisation was set to `True`
