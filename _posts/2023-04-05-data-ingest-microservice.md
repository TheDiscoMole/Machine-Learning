---
layout: post
title: "Machine Learning Pipeline (GCP) Part 1: Data Ingestion Microservice"
date: 2023-04-05
excerpt: "A generic data ingestion GCP microservice example using weather data"
tag:
comments: false
---

Project description can be found [HERE](https://thediscomole.github.io/portfolio/machine-learning-pipeline/)

This post describes the implementation basics required to write a cost-effective, high-uptime data ingestion microservice on the Google Cloud as part of a Machine Learning pipeline. The example will be based around consuming free weather APIs and is used to aggregate forecast data.

Code for this post can be found **[HERE](https://github.com/TheDiscoMole/example-pipe/ingest)**

------------------------------------------------------------------

## Architecture

Our goals for this service is to make a cost-effective & scalable service that can handle data consumption efficiently. Cost of services in the cloud generally revolve around 3 metrics:

1. Memory Usage
2. vCPU Uptime
3. Network Usage

We will be writing the service in Go to help with the optimization for the first 2 points. Go is probably the best language for any microservice that doesn't specifically require another language for library and/or tool support. Its C-like compiler means that its almost as fast and memory efficient as most custom C++ code, whilst it's goroutine-nature means that asynchronous execution is handled just as well as Javascript.

For the example of consuming free weather APIs we will be reaching API limits before we even get close to fully utilizing the uptime of a single compute resource in the cloud. We can improve our efficiency even more by creating timed-event triggers to consume forecasts until we reach our API limits using Cloud Run (serverless).

Our code structure follows most standard deep Go application file structures.

```
├── build
├── cmd
│   └── server
├── config
├── internal
│   ├── server
│   │   └── router
│   └── service
│   │   ├── weather
│   │   │   ├── api     # data consumers for api sources
│   │   │   └── ...     # weather consumer implmentation
│   │   └── ...         # other data source types
├── pkg
│   ├── model           # models for unified data types
│   ├── repository      # data store repositories
│   └── ...
├── go.*
├── Dockerfile
└── cloudbuild.yaml
```

------------------------------------------------------------------

## Implementation

**Deployment**: Our Cloud Run service requires us to containerize the application. Docker makes this easy using `Dockerfile`s and `docker build`:

```dockerfile
FROM golang:1.20-alpine as builder

WORKDIR /app
COPY go.* ./

RUN go mod download
COPY . ./

RUN go build -C cmd/server -v -o ../../build/server
CMD ["./build/server"]
```

**Event**: We can set up [scheduled HTTP events](https://cloud.google.com/run/docs/triggering/using-scheduler) to trigger our Cloud Run instance to consume the next batch API data on regular intervals.

**Server**: Since we will be consuming very simple HTTP requests that are just triggers to start up our data consumption, our request handler should be as simple and light weight as possible. [Chi](https://go-chi.io/#/) is a great library for this. Here is a sample of all we need to set up our server and manage our data type ingest triggers/routes.

`main.go`
```go
func main () {
    ...

    server := chi.NewRouter()
    server.Use(middleware.Logger)
    server.Post("/weather", router.Weather(configs))

    http.ListenAndServe(":" + configs.Port, server)
}
```

`internal/server/router/weather.go`
```go
func Weather (configs *config.Config) func (http.ResponseWriter, *http.Request) {
    return func (w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        client := weather.NewClient(configs)

        if err := client.Forecast(ctx); err != nil {
            fmt.Println(err)
            w.WriteHeader(500)
        }
    }
}
```

We don't really care about authentication as this will be handled automatically through the communication protocol settings in our private cloud.

**Service**: The service is responsible for the meat of our application. It implements the interface for the consumption & normalization of the datatype that we are consuming. In regards to our event-driven weather ingest, this means we need to handle batch requests GET requests for every API we will be consuming and mapping the responses to our unified data model located under `pkg/model`. We achieve this through an implementation of the following samples:

`internal/service/weather/api/[API_TYPE]/forecast.go`
```go
func (c *Client) ForecastLocation (coordinate model.Coordinate) ([]*model.Forecast, error) {
    url := c.baseUrl + "/forecast" + c.urlArgs

    // get apiType forecast
    response, err := c.client.Get(url)

    if err != nil {
        return nil, err
    }
    defer response.Body.Close()

    // validate status code
    if response.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("API_TYPE Status Code not OK: %d", response.StatusCode)
    }

    // decode forecast
    forecastResponse := apiTypeResponse{}

    if err := json.NewDecoder(response.Body).Decode(&forecastResponse); err != nil {
        return nil, err
    }

    // convert to unified model
    return forecastResponse.toForecasts(), nil
}
```

`internal/service/weather/api/[API_TYPE]/model.go`
```go
type apiTypeResponse{
    ...
}

func (f apiTypeResponse) toForecasts() []*model.Forecast {
    // map apiTypeResponse onto a slice of forecasts used by our internal schema
}
```

After consuming the forecast into the `apiTypeResponse` json model specified by the API provider, we convert it into our unified model schema, specified in `pkg/model/weather.go`, using `forecast.toForecasts()`. Now we are ready to save these weather forecasts as though they stem from a single source.

**Storage**: Finally we would, normally, be implementing the data storage using the repository pattern for layer abstraction and to simplify injections for our testing library. This would be accomplished by defining the repository interface under `pkg/repository` and implementing them in each consumer type (eg `internal/service/weather/repository`). As this sample service is part of a Machine Learning pipeline in the Google Cloud we will using a single generalizable method for storage and task forwarding.

Many API source we may consider consuming could be full-uptime data streams, such as the recently deprecated Twitter Sample Stream (the original subject of this ML pipeline example, until Elon happened). It is for high-demand streams such as this that ingestion and pre-processing separation was invented, as both consuming and pre-processing such data streams can becomes a bottle-necking nightmare. This problem can solved by saving data batches to a file storage system and queueing processing jobs for workers to consume. Cloud Pub/Sub and Storage are perfect for this. Cloud storage becomes our offloading file system and Pub/Sub will act as the fully managed task queue that interfaces task forwarding, through event triggers, perfectly.

We store our data and publish our tasks as follows:

`pkg/repository/storage.go`
```go
func (s *Storage) Save (ctx context.Context, filename string, data []byte) error {
    client, err := storage.NewClient(ctx)

    if err != nil {
        return err
    }
    defer client.Close()

    // create object writer
    bucket := client.Bucket(s.Bucket)
    object := bucket.Object(filename)
    writer := object.NewWriter(ctx)

    // write data
    if _, err := writer.Write(data); err != nil {
        return err
    }
    if err := writer.Close(); err != nil {
        return err
    }

    return nil
}
```

`pkg/publish/publish.go`
```go
func (c *Client) Publish (ctx context.Context, topic string, data string, attributes map[string]string) error {
    // create Google PubSub client
    client, err := pubsub.NewClient(ctx, c.projectID)

    if err != nil {
        return err
    }
    defer client.Close()

    // publish message
    publisher := client.Topic(topic)
    publishResult := publisher.Publish(
        ctx,
        &pubsub.Message{
            Data: []byte(data),
            Attributes: attributes,
        },
    )

    // handle failure
    if _, err := publishResult.Get(ctx); err != nil {
        return err
    }

    return nil
}
```

------------------------------------------------------------------

Continue reading about the implementation details of my example pipeline in Part 2: [Data Pre-Processing GCP Microservice (weather API example)](https://thediscomole.github.io/portfolio/data-preprocessing-microservice/)
