---
layout: post
title: "Machine Learning Pipeline (GCP) Part 3: Prediction Microservice"
date: 2023-04-05
excerpt: "A generic ML model training GCP microservice example using weather data"
tag:
comments: false
---

Project description can be found **[HERE](https://thediscomole.github.io/portfolio/machine-learning-pipeline/)**<br/>
Part 2 can be found **[HERE](https://thediscomole.github.io/portfolio/data-preprocessing-microservice/)**

This post describes the implementation basics required to write a cost-effective, high-uptime data prediction microservice on the Google Cloud as part of a Machine Learning pipeline. The example will implement a weather prediction model, whilst adhering to some generic design priciples such that the service can be reused for any and all model training we wish to perform.

Code for this post can be found **[HERE](https://github.com/TheDiscoMole/example-pipe/tree/main/predict)**

------------------------------------------------------------------

## Architecture

The model will be implmented using Pytorch and deployed to the Google Vertex AI Cloud using a Docker container for the custom training process and the automatic inference-deploy feature for our pretrained model.

To generalize the model trainer and allow for it to train a plethera of differnt models we will adhere to following file structure:

```
├── model
│   │   ├── dataset
│   │   │   ├── weather.py
│   │   │   └── ...
│   │   ├── modules
│   │   │   ├── attention.py
│   │   │   └── ...
│   │   └── repository
│   │   │   ├── model.py
│   │   │   └── ...
│   ├── weather.py
│   └── ...
├── main.py
├── Dockerfile
└── cloudbuild.yaml
```

------------------------------------------------------------------

## Implementation

**Deployment**: By modifying our `cloudbuild.yaml` to instead override our latest image, by always giving it the same name, we effectivly achieve pseudo-CI/CD for our model trainer:

```yaml
steps:
  # Build the container image
  - name: 'gcr.io/cloud-builders/docker'
    args:
    - 'build'
    - '-t'
    - 'gcr.io/$PROJECT_ID/train:$COMMIT_SHA'
    - '.'
    dir: 'train'

  # Push the container image to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
    - 'push'
    - 'gcr.io/$PROJECT_ID/train:latest'
```

**Dataset**: The dataset we extracted from our ingest service in [Part 1](https://thediscomole.github.io/portfolio/data-ingest-microservice/) contains the following weather features:

* time
* latitude
* longitude
* temperature
* apparent temperature
* pressure
* humidity
* cloudiness
* visibility
* precipitation probability
* rain volume
* snow volume
* wind speed
* wind angle
* wind speed

By extending the `torch.utils.data.Dataset` class we can create a dataset that our Pytorch library can fully manage and turn into training & validation batches for us:

```py
class Weather (torch.utils.data.Dataset):
    def __init__ (self):
        # connect to bigquery
        client = bigquery.Client()

        # load dataset
        data = client.query("SELECT * from weather")
        data = data.to_dataframe()

        # store dataset and tensor column selection
        self.data = data
        self.tensor_columns = [
            "latitude_sin", "latitude_cos",
            "longitude_sin", "longitude_cos",
            "week_of_the_year_sin", "week_of_the_year_cos",
            "day_of_the_week_sin", "day_of_the_week_cos",
            "temperature", "temperature_feels_like",
            "pressure", "humidity", "cloudiness", "visibility",
            "precipitation_probability", "rain_volume", "snow_volume",
            "wind_speed", "wind_angle", "wind_speed"
        ]

        # override len(data)
        def __len__ (self):
            return len(self.data)

        # override data[idx]
        def __getitem__ (self, idx):

            location = torch.tensor(self.data.iloc[idx][self.tensor_columns[:8]].values)
            history = self.locationHistory(forecast["latitude"], forecast["longitude"], 100, forecast["time"], forecast["time"] - 60*60*24*7)
            forecast = torch.tensor(self.data.iloc[idx][self.tensor_columns[8:]].values)

            return location, history, forecast

        # weather history for a location + radius
        def locationHistory (self, latitude, longitude, radius, time_start, time_stop):
            ...
```

**Model**: Though transformers are a bit flavour of the month right now, most of their principlis actually work very well with weather contextualization and prediction. It stands to reason that encoding the previous weather conditions, in the area, would allow a model to make better predictions. We define this history context in the above `locationHistory` function to be the area around geo location to be predicted, during a certain period. Now each training sample will consist of:

* location: the encoding for the geo-location and datetime of the forecast
* history: the weather conditions in a 100km "radius" around the geo-location for the last week
* forecast: the next forecast for the geo-location (our target)

We want to encode the history data and decode the location request, using the history encoding as context, to predict the forecast. Whilst this sounds very transformer-like, an exact copy of the standard transformer model would not work for us. These models come with something called positional-encoding baked in for sequence prediction. Our data does not come in a chronological oder, it's location depends on the space-time which we have already encoded during pre-processing. Therefore we will be implementing a simplification of the standard transformer model for our own needs.

For the sake of our weather example, our model uses a simple attention encoder:

```py
class AttentionEncoder (torch.nn.Module):
    def __init__ (self, model_dim,
        nhead=8,
        activation=torch.nn.ReLU,
        dropout=1e-1,
        layer_norm_eps=1e-5
    ):
        self.dropout = torch.nn.Dropout(dropout)
        self.activation = activation

        self.attention = torch.nn.MultiheadAttention(model_dim, nhead, dropout=dropout)
        self.normalize1 = torch.nn.LayerNorm(attention_dim, eps=layer_norm_eps)
        self.linear1 = torch.nn.Linear(model_dim, model_dim)
        self.normalize2 = torch.nn.LayerNorm(model_dim, eps=layer_norm_eps)

    def forward (self, key, query, value):

        attention = self.attention(key, query, value)
        attention = self.dropout(attention)

        output = self.normalize1(input + attention)
        output = self.normalize2(output + self.linear2(output))

        return self.activation(output)
```

This encoder is used in both the encoding and decoding step to generate our simple weather model:

```py
class Weather (torch.nn.Module):
    def __init__ (self,
        dropout=1e-1,
    ):
        self.dropout = torch.nn.Dropout(dropout)

        self.linear1 = torch.nn.Linear(18, 512)
        self.encoder = modules.AttentionEncoder(512)
        self.linear2 = torch.nn.Linear(8, 512)
        self.decoder = modules.AttentionEncoder(512)
        self.linear3 = torch.nn.Linear(512, 10)

    def forward (self, location, history):

        # encode the historical data using self-attention
        encode = self.linear1(history)
        encode = self.dropout(encode)
        encode = self.encoder(encode, encode, encode)

        # decode the forecast using the historical weather data and the location/time we wish to predict.
        decode = self.linear2(location)
        decode = self.dropout(decode)
        decode = self.decoder(encode, encode, decode)

        return self.linear3(decode)
```

**Training**: Because the goal for the microservice is to be able serve as the trainer for all the models we may wish to train on any of the data we are ingesting, we override the `torch.nn.Module.train` function with all the training steps for our end models. The models semselves are responsible to initiating datasets, training and saving of trained `state_dict`s:

```py
    def train (self, device,
        batch_size=args.batch_size,
        learning_rate=args.learning_rate,
        learning_rate_decay=args.learning_rate_decay,
        learning_rate_decay_step_size=args.learning_rate_decay_step_size,
        momentum=args.momentum,
        weight_decay=args.weight_decay
    ):
        self.to(device)
        self.training = True
        for module in self.children(): module.train(self.training)

        # set up dataset
        dataset = dataset.Weather()

        training_set, validation_set = torch.utils.data.random_split(dataset, [len(dataset) - len(dataset) / 9, len(dataset) / 9])
        training_loader, validation_loader = torch.utils.data.DataLoader(training_set, batch_size=batch_size, shuffle=True), torch.utils.data.DataLoader(validation_set, batch_size=batch_size)

        # prepare optimizer
        criterion = torch.nn.MSELoss().to(device)
        optimizer = torch.optim.SGD(model.parameters(), learning_rate, momentum=momentum, weight_decay=weight_decay)
        scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=learning_rate_decay_step_size, gamma=learning_rate_decay)

        # remember losses
        losses = []
        best = 1.

        # run training
        for epoch in range(epochs):
            for batch, (location, history, target) in enumerate(training_loader):
                location = location.view(-1, 1, 8)
                target = target.view(-1, 1, 10)

                location = location.to(device=device, dtype=torch.bfloat16, non_blocking=True)
                history = history.to(device=device, dtype=torch.bfloat16, non_blocking=True)
                target = target.to(device=device, dtype=torch.bfloat16, non_blocking=True)

                output = model(location, history)
                loss = criterion(output, target)

                optimizer.zero_grad()
                loss.backward()
                optimizer.step()

            losses.append(0.)
            for batch, (location, history, target) in enumerate(validation_loader):
                with torch.no_grad():
                    location = location.to(device=device, dtype=torch.bfloat16, non_blocking=True)
                    history = history.to(device=device, dtype=torch.bfloat16, non_blocking=True)
                    target = target.to(device=device, dtype=torch.bfloat16, non_blocking=True)

                    output = model(location, history)
                    losses[-1] += criterion(output, target)

            print(f'epoch: {epoch} loss: {losses[-1]}')

            # save if best checkpoint
            if losses[-1] < best:
                repository.model.save({
                    "epoch": epoch,
                    "model_state_dict": model.model_state_dict(),
                    "loss": loss
                }, "model/weather.pt")
                best = losses[-1]

            scheduler.step()
```

**Prediction**: Finally we can deploy our models using the Vertex AI GCP interface and use our google credentials to to make requests to our trained models:

```py
from google.cloud import aiplatform

def predict(project, location, inputs, endpoint):
    aiplatform.init(project=project, location=location)

    endpoint = aiplatform.Endpoint(endpoint)
    prediction = endpoint.predict(instances=inputs)

    return prediction
```
