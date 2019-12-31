---
layout: post
title: Building a data pipeline with Spring Cloud Data Flow to detect objects on images from Twitter
tags: [spring, spring-cloud-data-flow]
---

In this post I introduce you to the creation of powerful data pipelines with Spring Cloud Data Flow. The pipeline which we build consumes streaming data from Twitter, parses the image urls from the tweets, downloads the images with an HTTP client, detects objects on the images by Tensorflow, saves the images (annotated by bounding boxes of the detected objects) on the file system and stores the labels in MySQL. Spring Cloud Data Flow provides the possibility to build this pipeline normally without any programming, but in this case actually I had to write some code, because I discovered two bugs in the app.

## Prerequisites

You need to have 
- Spring Cloud Data Flow Server
- MySQL >=5.7.8 (up to 5.7.8 it supports JSON data type)
- Apache Kafka
- Spring Cloud Skipper (I suggest you to follow [this](https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/#getting-started) guide to install these)
- [Cloud Data Flow Shell](https://dataflow.spring.io/docs/installation/local/docker/#shell)

installed. As well as you need to have a [Twitter application](https://developer.twitter.com/). I installed them on a Kubernetes cluster, so some part of the solution is based on that.

## Building the pipeline

I used the default apps which are provided by the Spring Cloud Stream App Starters, on the following image you can see the data pipeline exported from the Data Flow UI.

![Spring Cloud Data Flow pipeline](/img/posts/spring-cloud-data-flow-data-pipeline.jpg "Spring Cloud Data Flow pipeline"){: .center-block :}

The pipeline can be created by two commands, let's see the first one.

{: .no-break}
```bash
stream create twitter --definition 
    "twitterstream 
        --twitter.credentials.consumer-key=<your-consumer-key> 
        --twitter.credentials.consumer-secret=<your-consumer-secret> 
        --twitter.credentials.access-token=<your-access-token> 
        --twitter.credentials.access-token-secret=<your-access-token-secret> 
        --twitter.stream.track=technology | 
    splitter 
        --expression=#jsonPath(payload,'$.entities.media[*].media_url') | 
    httpclient 
        --httpclient.http-method=GET 
        --httpclient.url-expression='new String(payload)' 
        --httpclient.headers-expression={'Content-Type':'application/octet-stream',
                                         'url':'new String(payload)'} 
        --httpclient.expected-responseType=byte[] | 
    object-detection-str 
        --tensorflow.mode=header | 
    file 
        --file.directory=images 
        --file.name-expression=headers[url].substring(headers[url].lastIndexOf('/')+1)
    " --deploy
```

### Twitter stream source

This app ingests data from Twitter, you have to replace `<your-consumer-key`, `<your-consumer-secret>`, `<your-access-token>` and `<your-access-token-secret>` in the above command with your own. By setting a list of comma separated values to the `--twitter.stream.track` option you can [influence](https://developer.twitter.com/en/docs/tweets/filter-realtime/guides/basic-stream-parameters) the topic of the images. The output of this source is a JSON which represents a tweet.

```
{ 
   "created_at":"Fri Dec 20 23:20:42 +0000 2019",
   "id":1208165358114345000,
   "id_str":"1208165358114344960",
   "source":"NiceMagazine",
   ...
   "user":{ 
      "id":897870811670904800,
      "id_str":"897870811670904832",
      "name":"NiceMagazine",
      ...
   },
   "entities":{ 
      ...
      "media":[ 
         { 
            "media_url":"http://pbs.twimg.com/media/EMRD_YCWoAQpsAx.jpg",
            ...
         }
   }
}
```

### Splitter processor

The splitter processor parses the input JSON and based on the `--expression` option extracts information or parts of the JSON and forwards every occurence to the output queue. The value of the `--expression` option is a SpEL expression, in this case it's `#jsonPath(payload,'$.entities.media[*].media_url')` which takes the `media` array in the JSON and extracts the value of every occurence of the `media_url` property. The output of the splitter processor is a URL.

### HTTP client processor

The HTTP client processor sends a request to a URL and sends the response payload of the request to its output queue. The HTTP method can be configured by the `--httpclient.http-method` option and we set it to `GET`, because we want to download the image from Twitter. The request URL can be set by the `--httpclient.url-expression` option and now it's `'new String(payload)'`, since the input message is a byte array, therefore we have to transform it to a string. We have to change the `Content-Type` header of the output message, because its default value is `application/json`, but the object detection app requires an `application/octet-stream`. We also add the URL of the image to the header, because later we would like to store it in MySQL, that's why we set the `--httpclient.headers-expression` option to `{'Content-Type':'application/octet-stream','url':'new String(payload)'}`. The last option we have to set is the `--httpclient.expected-responseType`, its default value is `String.class`, now it should be `byte[]`.

### Object detection processor

The object detection processor detects objects on the received images, creates a new image annotated by bounding boxes and puts the new image and the labels of the detected objects to the queue. We have to set `--tensorflow.mode` to `header`, because we would like to send the annotated image as the payload and the labels in the header.

So far it's seems to be pretty easy, but if you use the `object-detection-processor-kafka:2.1.1.RELEASE` docker image of the object detection processor or the earlier versions, it'll throw `java.lang.NoClassDefFoundError: Could not initialize class sun.font.SunFontManager` exception, because `sun.font.SunFontManager` is not part of the OpenJDK 8. As a workaround I created my own docker image and installed the `fontconfig` which solved the issue. You can add my docker image to your Spring Cloud Data Flow Server `docker:strsz/object-detection-processor-kafka:1.0.0` or you can build your own.

#### Build your own object detection processor

Follow the next steps:

```bash
git pull https://github.com/spring-cloud-stream-app-starters/tensorflow
cd tensorflow
mvn clean install -PgenerateApps
cd apps/object-detection-processor-kafka
mvn clean package docker:build
cd target/docker/springcloudstream/object-detection-processor-kafka/build/
```

Open the `DockerFile` and insert `RUN apt-get update && apt-get install fontconfig -y` before the entrypoint.

```bash
docker login
docker build -t <hub-user>/object-detection-processor-kafka:1.0.0 .
docker push <hub-user>/object-detection-processor-kafka:1.0.0
```

Now you can add your own docker image to Spring Cloud Data Flow Server.

### File sink

The file sink app writes each message it receives to the file system, you can specify the name of the directory by setting `--file.directory` to `images`. In this case the filename is parsed from the url header and set by the `--file.name-expression=headers[url].substring(headers[url].lastIndexOf('/')+1)` option.

### JDBC sink

The JDBC sink writes its incoming message to an RDBMS using JDBC, you can set the table name with the `--jdbc.table_name` option to `image_labels` and the value mapping to the columns by setting the `--jdbc.columns` to `url:headers[url],label:headers[result]`. The value of the `headers[url]` header is the URL which we set in the HTTP client processor, the `headers[result]` was added to the headers by the object detection processor and it consists of the labels of the detected objects in JSON. You can specify the JDBC driver by the `--spring.datasource.driver-class-name` option, the URL of your MySQL by `--spring.datasource.url`, the username by `--spring.datasource.username` and your password by `--spring.datasource.password`.

I think it's a bug, but the `jdbc-sink-kafka:2.1.3.RELEASE` doesn't support the mapping of header values to columns, so I applied the following minor modification on the codebase and now it works fine with the headers. To add this change to your docker image you have to build your own with the same method which I described earlier or you can use my image `strsz/jdbc-sink-kafka:1.0.0` and add it to your Spring Cloud Data Flow Server.

{: .no-break}
```java
diff --git a/spring-cloud-starter-stream-sink-jdbc/src/main/java/org/springframework/cloud/stream/app/jdbc/sink/JdbcSinkConfiguration.java b/spring-cloud-starter-stream-sink-jdbc/src/main/java/org/springframework/cloud/stream/app/jdbc/sink/JdbcSinkConfiguration.java
index 90b64f6..ce8b509 100644
--- a/spring-cloud-starter-stream-sink-jdbc/src/main/java/org/springframework/cloud/stream/app/jdbc/sink/JdbcSinkConfiguration.java
+++ b/spring-cloud-starter-stream-sink-jdbc/src/main/java/org/springframework/cloud/stream/app/jdbc/sink/JdbcSinkConfiguration.java
@@ -160,7 +161,7 @@ public class JdbcSinkConfiguration {
- convertedMessage = new MutableMessage<>(messageStream.collect(Collectors.toList()));
+ convertedMessage = new MutableMessage<>(messageStream.collect(Collectors.toList()), message.getHeaders());
```

Before you build the second stream you have to create a database, use the following SQL commands:

```sql
CREATE DATABASE twitter;
USE twitter;
CREATE TABLE image_labels(
	id int NOT NULL AUTO_INCREMENT,
	url varchar(500),
	label json,
	PRIMARY KEY (id)
);
```

This stream connects our deployed object detection processor to the JDBC sink.

```bash
stream create mysql --definition 
    ":twitter.object-detection-str > 
    jdbc-header 
        --jdbc.table_name=image_labels 
        --jdbc.columns=url:headers[url],label:headers[result] 
        --spring.datasource.driver-class-name=org.mariadb.jdbc.Driver 
        --spring.datasource.url='jdbc:mysql://<mysql-ip-address>/twitter' 
        --spring.datasource.username=<your-username> 
        --spring.datasource.password=<your-password>
    " --deploy
```

## Results

![Cat detected by Tensorflow](/img/posts/tensorflow-cat.jpg "Cat detected by Tensorflow"){: .center-block :}

```json
{
    "x1": 0.14578229,
    "x2": 0.93025094,
    "y1": 0.084840715,
    "y2": 0.5899739,
    "cid": 17,
    "name": "cat",
    "confidence": 0.9662085
}
```