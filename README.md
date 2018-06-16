## Prepare Fluentd image with your Config and Plugin.

Prepare Fluentd’s configuration file `fluentd/conf/fluent.conf`. in_forward plugin is used for receive logs from Docker logging driver, and out_elasticsearch is for forwarding logs to Elasticsearch.

```
# fluentd/conf/fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<match *.**>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
    flush_interval 1s
  </store>
  <store>
    @type stdout
  </store>
</match>
```
Prepare `fluentd/Dockerfile` with the following content, to use Fluentd’s official Docker image and additionally install Elasticsearch plugin.
```
# fluentd/Dockerfile
FROM fluent/fluentd:v0.12-debian
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.2"]
```
## Create `docker-compose.yaml` With the YAML file below, you can create and start all the services.
```
version: '2'
services:
  web:
    image: httpd
    ports:
      - "80:80"
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: httpd.access

  fluentd:
    build: ./fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    links:
      - "elasticsearch"
    ports:
      - "24224:24224"
      - "24224:24224/udp"

  elasticsearch:
    image: elasticsearch
    expose:
      - 9200
    ports:
      - "9200:9200"

  kibana:
    image: kibana
    links:
      - "elasticsearch"
    ports:
      - "5601:5601"
```
## Start `docker-compose` using the `docker-compose.yaml`.
```
$ docker-compose up -d 
```
List the containers running.
```
$ docker container ls

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                          NAMES
533709d98baf        httpd               "httpd-foreground"       14 seconds ago      Up 11 seconds       0.0.0.0:80->80/tcp                                             elk_web_1
a41e7f1dee71        kibana              "/docker-entrypoin..."   17 seconds ago      Up 14 seconds       0.0.0.0:5601->5601/tcp                                         elk_kibana_1
fdd7bd4cd5e6        elk_fluentd         "/bin/entrypoint.s..."   17 seconds ago      Up 15 seconds       5140/tcp, 0.0.0.0:24224->24224/tcp, 0.0.0.0:24224->24224/udp   elk_fluentd_1
73ba268055ea        elasticsearch       "/docker-entrypoin..."   19 seconds ago      Up 17 seconds       0.0.0.0:9200->9200/tcp, 9300/tcp                               elk_elasticsearch_1
```
## Generate httpd Access Logs.
Let’s access to `httpd` to generate some access logs.
```
$ curl http://localhost:80/
<html><body><h1>It works!</h1></body></html>
```
## Access kibana
- Please go to `http://localhost:5601/` with your browser. Then, you need to set up the index name pattern for Kibana. Please specify `fluentd-*` to Index name or pattern and press `Create` button.

- Then, go to `Discover` tab and take a look at the logs. As you can see, logs are properly collected into `Elasticsearch + Kibana`, via `Fluentd`.

At `Kibana` add Filters from the `Available Fields` Add Filters like `container_id`, ` container_name`,`error`, `logs`, `message` and ` plugin_id`. 

Now at the terminal start few container with `--log-driver=fluentd`.
```
$ docker run --log-driver=fluentd --name demo1 ubuntu echo "Hello Fluentd!"
$ docker run --log-driver=fluentd --name demo2 busybox echo "Hello Fluentd!"
```
Open the kibana tab in browser and you wil see there logs of the perticular containers.

![Kibana Dashboard](https://raw.githubusercontent.com/vishalcloudyuga/Logging-with-Fluentd-ELK/master/Kibana.png)


