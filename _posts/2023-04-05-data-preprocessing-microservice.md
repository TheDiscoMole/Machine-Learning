---
layout: post
title: "Machine Learning Pipeline (GCP) Part 2: Data Pre-Processing Microservice"
date: 2023-04-05
excerpt: "A generic data pre-processing GCP microservice example using weather data"
tag:
comments: false
---

Project description can be found [HERE](https://thediscomole.github.io/portfolio/machine-learning-pipeline/)<br/>
Part 1 can be found [HERE](https://thediscomole.github.io/portfolio/data-ingest-microservice/)

This post describes the implementation basics required to write a cost-effective, high-uptime data pre-processing microservice on the Google Cloud as part of a Machine Learning pipeline. The example will implement a generic interface for the pre-processing of several data types.

Code for this post can be found **[HERE](https://github.com/TheDiscoMole/example-pipe/preprocess)**

------------------------------------------------------------------

## Architecture

As several real-world data types have existing/pre-trained processing strategies/models it makes sense to work with those data types in Python. Python has by far the most mature community and robust libraries/tools in the fields of Machine Learning and Data Science. As discussed in in the previous part, we could write several pre-processing services in different languages to improve cloud utilization, but for the sake of example-simplicity the entire service will be a single Python application.

This service will be triggered by batches of pre-processing tasks we deliver from our ingestion service(s) through Pub/Sub messages. Therefore almost every reason for architecture decisions from Part 1 apply here as well. Cloud Run is an excellent solution for the processing of infrequent data loads.

We will be using a simple Python API server file structure to handle the [Pub/Sub HTTP triggers](https://cloud.google.com/run/docs/tutorials/pubsub):

```
├── build
├── middleware
│   └── pubsub.py
├── repository
├── service
│   │   └── model
│   ├── audio.py
│   ├── image.py
│   └── text.py
├── main.py
├── Dockerfile
└── cloudbuild.yaml
```

------------------------------------------------------------------

## Implementation

**Deployment**: Our Cloud Run service requires us to containerize the application. Docker makes this easy using `Dockerfile`s and `docker build`:

```dockerfile
# Use the official Python image.
# https://hub.docker.com/_/python
FROM python:3.12-slim

# Allow statements and log messages to immediately appear in the Cloud Run logs
ENV PYTHONUNBUFFERED True

# Copy application dependency manifests to the container image.
# Copying this separately prevents re-running pip install on every code change.
COPY requirements.txt ./

# Install production dependencies.
RUN pip install -r requirements.txt

# Copy local code to the container image.
WORKDIR /app
COPY . ./

# Run the web service on container start-up.
# Timeout is set to 0 to disable the timeouts of the workers to allow Cloud Run to handle instance scaling.
CMD exec gunicorn --bind :$PORT --workers 1 --max_requests 80 --timeout 0 main:app
```

**Server**: Since we will be consuming very simple HTTP requests that are just triggers to start up our data processing tasks, our request handler should be as simple and light weight as possible. [Flask](https://flask.palletsprojects.com/en/2.2.x/) is a great library for this. Here is a sample of all we need to set up our server and manage triggers/routes.

`main.py`
```py
from flask import Flask, request
from middleware import pubsub
from service import audio, text, image

app = Flask(__name__)
app.wsgi_app = pubsub(app.wsgi_app)

// route handler
@app.route("/route", methods=["POST"])
async def route_handler ():
    # do stuff
```

This server will only be responsible for resolving Pub/Sub tasks, so it makes sense to implement the input validation as a middleware that validates and deconstructs input before every route.

`middleware/pubsub.py`
```py
from werkzeug.wrappers import Request, Response

# middleware to validate pubsub request format
class pubsub ():
    def __init__ (self, app):
        self.app = app

    def __call__ (self, environ, start_response):
        request = Request(environ)
        message = request.get_json()

        if not message:
            response = Response("received message not json formatted", mimetype= 'text/plain', status=400)
            return response(environ, start_response)

        if not isinstance(message, dict) or "message" not in message:
            response = Response("received message not pub/sub formatted", mimetype= 'text/plain', status=400)
            return response(environ, start_response)

        environ["filename"] = message["data"]
        environ["attributes"] = message["attributes"]

        return self.app(environ, start_response)
```

We don't really care about authentication as this will be handled automatically through the communication protocol settings in our private cloud.

**Service**: We will be implementing pre-processing for 3 example data types:

* audio: A standard approach for feature extraction from wave functions is to convert the sample from the Time-Domain to normalized slices of the Frequency-Domain on the Mel scale, called [Mel-Spectrograms](https://towardsdatascience.com/getting-to-know-the-mel-spectrogram-31bca3e2d9d0). When converted to Decibel scale, this creates a feature-wise dense representation of the wave form, which is proven to be quite effective for Machine Learning.

```py
import librosa
import numpy as np

from ..repository import storage

# mel-transform audio file
async def preprocess (filename):
    async with storage.open(f'ingest/{filename}', 'r') as file:
        y, sr = librosa.load(file)

    # compute mel-transform
    mel = librosa.feature.melspectrogram(y=y, sr=sr, n_fft=2048, hop_length=512, n_mels=128)
    mel = librosa.power_to_db(mel, ref=np.max)

    async with storage.open(f'preprocess/{filename}', 'w') as file:
        np.save(file, mel)
```

* images:

* text:

**Storage**:

------------------------------------------------------------------

Continue reading about the implementation details of my example pipeline in Part 3: [Data Pre-Processing GCP Microservice (weather API example)](https://thediscomole.github.io/portfolio/prediction-microservice/)
