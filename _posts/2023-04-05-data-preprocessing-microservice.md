---
layout: post
title: "Machine Learning Pipeline (GCP) Part 2: Data Pre-Processing Microservice"
date: 2023-04-05
excerpt: "A generic data pre-processing GCP microservice example using audio/image/text data"
tag:
comments: false
---

Project description can be found **[HERE](https://thediscomole.github.io/portfolio/machine-learning-pipeline/)**<br/>
Part 1 can be found **[HERE](https://thediscomole.github.io/portfolio/data-ingest-microservice/)**

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

* images: Images come in many different sizes and formats, but our Machine Learning models will expect a consistent shape and format for modelling. We take our image input and resize the shortest pixel dimension to the expected size, whilst conserving aspect ratio, and save the image as the more efficient JPEG format. The reason, resizing across the shortest pixel dimension work, is because at model runtime we will be introducing noise to our images during training. This noise comes in the form of grey-scaling, exposure, image rotation and most importantly image subsampling. Even-though the largest of our images my not be appropriate for our model, since we will be subsampling our images anyway, across batches and training epochs the model will eventually see and learn from the entire image anyway. (there are some caveats with images with extremely differing dimensions, those should be pruned from the training data in a fully exhaustive pre-processing service)

```py
from PIL import Image

from ..repository import storage

# resize image file
async def preprocess (filename):
    async with storage.open(f'ingest/{filename}', 'r') as file:
        image = Image.open(file)

    # get dimensions
    width, height = image.size

    # shrink smallest dimension to 256 and other dimension with respect to aspect ratio
    if width > height: width, height = int(width * 256 / height), 256
    else: width, height = 256, int(height * 256 / width)

    # resize and format image
    image = image.resize((width, height))
    image = image.convert("RGB")

    async with storage.open(f'preprocess/{filename}', 'w') as file:
        image.save(file)
```

* text: Turning text into Machine Learning features (or text embedding) is done by converting substrings into feature vectors. This can be done at the word, sub-word and character level, where, for example, each character in the ASCII alphabet would correspond to a specific feature vector. This essentially emulates the neural network that would normally convert the character to a contextual representation and treats it as a learnable input. All three embedding levels come with their own challenges. Word embeddings are large and cumbersome vector spaces prone to issues with misspelled words and character embeddings are expensive on computation as each character gets a feature vector, instead of each word. Whilst sub-word embedding comes with its own issues its a healthy middle-ground that has been adopted by most state of the art text encoders. [BPEmb](https://bpemb.h-its.org/) uses an efficient byte-pair encoding strategy and as such has many light weight embedding models we can use to ensure the load on our pre-processor stays minimal. Most of the official BPEmb Python lib is useless to us and quite slow, but as we only wish to store the indexes of our sub-word embeddings we can use its underlying string encoder [SentencePiece](https://github.com/google/sentencepiece). Using the model files from BPEmb with SentencePiece instead speeds up our text embedding 8x! Unfortunately SentencePiece was written in C++ without and consideration for Python file pointers and we are forced to save our model inside our microservice, but its a trade-off worth making.

```py
import sentencepiece

from ..repository import storage

# embed text file
async def preprocess (filename):
    # load text embedding model
    spp = sentencepiece.SentencePieceProcessor(model_file='service/model/bpemb')

    async with storage.open(f'ingest/{filename}', 'r') as file:
        lines = file.readlines()

    # embed lines of text
    lines = [spp.encode(line) for line in lines]

    async with storage.open(f'preprocess/{filename}', 'w') as file:
        file.writelines(lines)
```

**Storage**: Our resultant pre-processed features and files can just be written right back to Google Storage, ready for loading by our models. The Python library for the Google Cloud implements Google Storage blobs quite nicely and allows us pass the blob around like a local file pointer. (see above pre-processors)

------------------------------------------------------------------

Continue reading about the implementation details of my example pipeline in Part 3: [Data Pre-Processing GCP Microservice (weather API example)](https://thediscomole.github.io/portfolio/prediction-microservice/)
