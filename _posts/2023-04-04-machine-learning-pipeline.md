---
layout: post
title: "Machine Learning Cloud Pipeline"
date: 2023-04-04
excerpt: "general purpose data ingestion/pre-processing and machine learning cloud pipeline"
project: true
tag:
comments: false
---

Often Machine Learning ideas/tests are hindered by availability/price of data and compute resources. Most of these issues can solved by deploying a continuous pipeline in the cloud.

This project goal is the implementation of a general purpose Machine Learning pipeline in terms of data aggregation, processing and predictions. The extend of this write-up will be limited to weather data modelling for simplicity, with parts of the explanations touching on the generalization of the pipeline that I will actually be running in my private cloud.

Code for this project can be found **[HERE](https://github.com/TheDiscoMole/example-pipe)**

------------------------------------------------------------------

## Project Architecture

**monolith or microservices?**: We will be optimizing for scale and cost efficiency in the Cloud with continuous and on-demand functionality it makes sense to split the pipeline into a subset of microservices. This will allow us to run on-demand or event based services on a serverless tech-stack and use programming languages appropriate for the functionality requirements at each part of the pipeline.

**mono-repo or multi-repo?**: Whilst multi-repository patterns can be far more convenient to manage CI/CD pipelines, with multiple repositories having one build trigger, I rather have my project isolated from other repositories on my GitHub. Additionally multi-repositories come with their own challenges for managing libraries required across multiple services. (such as proto definitions)

**AWS or GCP?**: Both cloud providers have essentially the same offering when it comes down to it. My personal preference lies with GCP due to its cleaner and more flushed out documentation and the existence of [Cloud Run](https://cloud.google.com/run). This is, to my most recent knowledge of serverless offerings, one of the only serverless solutions that can process multiple requests at once, allowing for even more efficient scaling in terms of compute cost.

**deployment**: Continuous deployment from a GitHub mono-repository to the Google Cloud is surprisingly simple using [Cloud Build](https://cloud.google.com/build), [Docker](https://www.docker.com/) and a `cloudbuild.yaml` configuration file tied to each microservice. Spin up an empty service in the Google Cloud user interface and use the deployment region and service name in your configuration file. Here is an example for Cloud Run: (change the final step if you wish to deploy the image to a different, relevant, cloud product)

```yaml
steps:
  # Build the container image
  - name: 'gcr.io/cloud-builders/docker'
    args:
    - 'build'
    - '-t'
    - 'gcr.io/$PROJECT_ID/<SERVICE_NAME>:$COMMIT_SHA'
    - '.'
    dir: '<SERVICE_DIR>' # change the working directory to the location of the microservice

  # Push the container image to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
    - 'push'
    - 'gcr.io/$PROJECT_ID/<SERVICE_NAME>:$COMMIT_SHA'

  # Deploy container image to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
    - 'run'
    - 'deploy'
    - '<SERVICE_NAME>'
    - '--image'
    - 'gcr.io/$PROJECT_ID/<SERVICE_NAME>:$COMMIT_SHA'
    - '--region'
    - '<SERVICE_REGION>'
images:
  - 'gcr.io/$PROJECT_ID/<SERVICE_NAME>:$COMMIT_SHA'
```

------------------------------------------------------------------

## implementation

For a complete Machine Learning pipeline, services can be categorized into the following general types:

1. **ingest microservice**: One of the most important parts of Machine Learning is data. Data aggregation in the forms of API consumption, Scraping and manual file uploads will cover most of the ways we will most likely feeding data into our pipeline. This group of services is responsible for the consumption of their assigned data source, and the mapping of the extracted data points into a unified format/schema, which is expected by services down stream. For the purposes of the weather prediction example we will be consuming forecasts from multiple free API providers and history dumps already provided by the Google Cloud. Post detailing implementation can be found **[HERE](https://thediscomole.github.io/portfolio/data-ingest-microservice/)**

2. **pre-processing microservice**: Data tends to messy and non-uniform even for data for similar sources. Even though the ingest services may have unified the format/schema, there are still data features which need to be normalized before we can feed them into our models. Images need have their sizes standardized, metrics like temperature need to be normalized, text features need to be embedded, corrupted data needs to be pruned, etc. The pre-processing services are responsible for this cleaning and data extraction. In the case of our weather pipeline pre-processing will be minimal, therefore the implemented example extend to other data types like audio, images and text. Post detailing implementation can be found **[HERE](https://thediscomole.github.io/portfolio/data-preprocessing-microservice/)**

3. **prediction microservice**: Here we implement our Machine Learning models and their training/prediction procedures. Our weather prediction example-model will consist of a Transformer. We encode our aggregated & unified features using a positional and temporal embedding and decode this weather context with the same positional/temporal embedding to get our prediction for the weather condition at the time and location of our choosing. Post detailing implementation can be found **[HERE](https://thediscomole.github.io/portfolio/prediction-microservice/)**
