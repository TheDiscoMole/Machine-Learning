---
layout: post
title: "Data Ingestion GCP Microservice (weather API example)"
date: 2023-04-05
excerpt: "A generic data ingestion GCP microservice example using weather data"
tag:
comments: false
---

This post describes the implementation basics required to write a cost-effective, high-uptime data ingestion microservice on the Google Cloud. The example will be based around consuming free weather APIs and is used to aggregate forecast data.

Code for this post can be found **[HERE](https://github.com/TheDiscoMole/pipeline/ingest)**

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
│   ├── server
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

**Event Triggers**: We can set up [scheduled HTTP events](https://cloud.google.com/run/docs/triggering/using-scheduler) to trigger our Cloud Run instance to consume the next batch API data on regular intervals.

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
