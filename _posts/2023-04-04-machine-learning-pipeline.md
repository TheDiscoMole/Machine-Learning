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

Code for this project can be found **[HERE](https://github.com/TheDiscoMole/pipeline)**

------------------------------------------------------------------

## Project Architecture

**monolith or microservices?**: We will be optimizing for scale and cost efficiency in the Cloud with continuous and on-demand functionality it makes sense to split the pipeline into a subset of microservices. This will allow us to run on-demand or event based services on a serverless tech-stack and use programming languages appropriate for the functionality requirements at each part of the pipeline.

**mono-repo or multi-repo?**: Whilst it can be far more convenient to manage CI/CD pipelines with multiple repositories having simple build triggers, I rather have my project isolated from other repositories on my GitHub. Additionally multi-repositories come with their own challenges for managing libraries required across multiple services. (such as proto definitions)

**AWS or GCP?**: Both cloud providers have essentially the same offering when it comes down to it. My personal preference lies with GCP due to its cleaner and more flushed out documentation and the existence of [Cloud Run](https://cloud.google.com/run). This is, to my most recent knowledge of serverless offerings, one of the only serverless solutions that can process multiple requests at once, allowing for even more efficient scaling in terms of compute cost.

**deployment**: Continuous deployment from a GitHub mono-repository to the Google Cloud is surprisingly simple using [Cloud Build](https://cloud.google.com/build), [Docker](https://www.docker.com/) and a `cloudbuild.yaml` configuration file tied to each microservice. Spin up an empty service in the Google Cloud user interface and use the deployment region and service name in your configuration file. Here is an example for Cloud Run: (change the final step if you wish to deploy the image to a different, relevant, cloud product)

{% highlight yaml %}
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
{% endhighlight %}


**microservices**:

1. **data ingestion**: One of the most important parts of Machine Learning is data. Data aggregation in the forms of API consumption, Scraping and manual file uploads will cover most of the ways we will most likely feeding data into our pipeline. For the purposes of the weather prediction example we will be consuming forecasts from multiple free API providers and history dumps already provided by the Google Cloud.
