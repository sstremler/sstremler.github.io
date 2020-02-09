---
layout: post
title: Fetching and storing air quality data with Spring Cloud Stream
tags: [spring, spring-cloud-stream]
---

Spring Cloud Stream is based on Spring Integration and can be used to create event-driven microservices by binding together loosely coupled applications which are communicating with each other through a message broker. In this post I provide an application which fetches data every hour from the [World Air Quality Index](https://waqi.info/), sends the data to a Kafka topic and stores the measured values in a database.

![Air quality data pipeline with Spring Cloud Stream](/img/posts/waqi-spring-cloud-stream.jpg "Air quality data pipeline with Spring Cloud Stream"){: .center-block :}

## Prerequisites

You need to have Apache Kafka running on localhost. For development purposes it can be set up easily in Docker. Save the following file as `docker-compose.yml` and type `docker-compose up -d` in the console to start it.

```
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

## Source

In the source application we send two types of requests to the World Air Quality Index API. The first one requests all the air quality monitoring stations in Germany, and the second one requests the measurement values (NO2, O3, SO2, PM10, PM2.5) of all the stations one by one. We identify German stations by setting the latitude and longitude bounding box values of the `GET` parameters to a proper value. We poll the stations every hour and that's why the `spring.cloud.stream.poller.fixed-delay` parameter has to be set to `3600000` ms. The data is sent to a Kafka topic called `sink`. Before we start the application we have to [register](https://aqicn.org/data-platform/token/#/) our own WAQI API token and set the value of the `air-quality.api.token` property.

```
spring.cloud.stream.poller.fixed-delay=3600000
spring.cloud.stream.bindings.stationSupplier-out-0.destination=sink

air-quality.api.parameter.countries.latlng={germany:'47.3024876979,5.98865807458,54.983104153,15.0169958839'}

air-quality.api.token=<your-token>
air-quality.api.base-url=https://api.waqi.info
air-quality.api.endpoints={stations:'/map/bounds/',city:'/feed/@{uid}/'}
```

The entrypoint of the application is the `stationSupplier` method. It has `@PollableBean` annotation which binds the return value of the method to the `sink` topic with the help of the `spring.cloud.stream.bindings.stationSupplier-out-0.destination` property. The `urlBuilderService.buildStationsUrl` method builds the request URL to get the station data. The `route` method sends the request to the selected URL. We have to set the return value as well which is `Station.class` in case of the station data. After receiving the response from the endpoint, we get the `data` property of the response with `map(response -> response.getData())`. The value of the `data` property is a JSON array. We iterate over it by calling `flatMap(Flux::fromIterable)`. Next we use the `uid` of the station to prepare our next REST call with the same methods we used before by calling `route(urlBuilderService.buildCountryUrl(station.getUid().toString()), Measurement.class)`. We get the `data` property again with `map(response -> response.getData())` and forward the measurement data of every station to Kafka.

{: .no-break}
```java
@PollableBean
public Supplier<Flux<Measurement>> stationSupplier() {
    return () -> route(urlBuilderService.buildStationsUrl(), Station.class).map(response -> response.getData())
            .flatMap(Flux::fromIterable)
            .flatMap(station -> route(urlBuilderService.buildCountryUrl(station.getUid().toString()), Measurement.class))
            .map(response -> response.getData())
            .flatMap(Flux::fromIterable);
}
```

Now let's have a look at the `route` method which we used above. This method sends a `GET` request to the selected endpoint provided by the `uriBuilderFunction`. Its return value can be configured to allow use of the `route` method for different requests.

{: .no-break}
```java
public <T> Flux<Response<T>> route(Function<UriBuilder, URI> uriBuilderFunction, Class<T> responseType) {
    return webClient.get().uri(uriBuilderFunction).retrieve()
            .bodyToFlux(ParameterizedTypeReference.forType(ResolvableType.forClassWithGenerics(Response.class, responseType).getType()));
}
```

The `UrlBuilderService` interface was used in the `stationSupplier` method previously. We can use it to build URLs to get the measurements and the stations in a country. The interface is implemented by the `DefaultUrlBuilderServiceImpl` class.

```java
@Service
public interface UrlBuilderService {
    public Function<UriBuilder, URI> buildStationsUrl();
    public Function<UriBuilder, URI> buildCountryUrl(String country);
}

public class DefaultUrlBuilderServiceImpl implements UrlBuilderService {

    ...

    public Function<UriBuilder, URI> buildStationsUrl() {
        return uriBuilder -> uriBuilder
                .path(this.endpoints.get("stations"))
                .queryParam("token", this.token)
                .queryParam("latlng", this.countries.get("germany"))
                .build();
    }

    public Function<UriBuilder, URI> buildCountryUrl(String country) {
        return uriBuilder -> uriBuilder
                .path(this.endpoints.get("city"))
                .queryParam("token", this.token)
                .build(country);
    }
}
```

The `Response<T>` class represents the response of the calls to the WAQI API endpoints. It always has a `status` and a `data` property, which can be an array or a single object. That's why the `@JsonFormat(with = JsonFormat.Feature.ACCEPT_SINGLE_VALUE_AS_ARRAY)` annotation is needed.

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class Response<T> {
    private String status;
    @JsonFormat(with = JsonFormat.Feature.ACCEPT_SINGLE_VALUE_AS_ARRAY)
    private List<T> data;
}
```

The `Station` POJO represents a station. The only important properties are the latitude, longitude and uid.

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class Station {
    private Double lat;
    private Double lon;
    private Integer uid;
    private String aqi;
    private String name;

    ...
}
```

The `Measurement` class contains the measured air quality values and the city where the measurement happened.

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class Measurement {
    @JsonIgnore
    private Integer aqi;
    private Integer idx;
    @JsonIgnore
    private City city;
    private Double no2;
    private Double o3;
    private Double p;
    private Double pm10;
    private Double pm25;
    private Double so2;
    private Double t;
    private Double w;
    private Double wg;
    private Date time;

    ...
}

@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class City {
    private String name;
    private String url;
    private Double[] geo = new Double[2];
}
```

## Sink

The sink application retrieves messages from the `sink` topic and stores the batched messages in a database. For development we use an H2 database which is configured in the `application.properties`.

```
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.url=jdbc:h2:mem:test
spring.datasource.username=sa
spring.datasource.password=

spring.cloud.stream.bindings.store-in-0.destination=sink
```

The `sink` topic is bound to the `store` bean. It consumes the messages and calls the `next` method of the `Batcher` class to batch the messages.

```java
@Bean
public Consumer<Flux<Measurement>> store(){
	return flux -> flux.subscribe(batcher::next);
}
```

The `Batcher` class uses a `UnicastProcessor`, which is a `Processor` implementation that takes a custom queue and allows only a single subscriber. The `listen` method tells the `processor` to collect the incoming messages in a `List`. `bufferTimeout` returns the `List` each time the buffer reaches a maximum size or the maximum time duration elapses. The `next` method gives the message to the `processor`.

```java
@Component
public class Batcher {

    private static final UnicastProcessor processor = UnicastProcessor.create();

    public void next(Measurement element) {
        processor.sink().next(element);
    }

    public Flux<List<Measurement>> listen() {
        return processor.bufferTimeout(10, Duration.ofSeconds(10));
    }

}
```

We have to subscribe to the batcher listener to save the messages. To do that we call the `saveAll` method of the `MeasurementRepository` interface. `MeasurementRepository` is just the child of the `CrudRepository` interface.

```java
@PostConstruct
public void subscribeListener() {
	batcher.listen().subscribe(measurementRepository::saveAll);
}

public interface MeasurementRepository extends CrudRepository<Measurement, Long> {
}
```

## Conclusion

We looked into how to create a producer and a consumer in Spring Cloud Stream, how to poll an external REST API and how to batch messages with Spring Reactor. The repository of the applications can be found on [GitHub](https://github.com/sstremler/air-quality-cloud-stream).