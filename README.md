# Kafka + ELK logging system

Run Elasticseach, Logstash, Kibana, Kafka, stack with Docker and Docker-compose.

Based on the images:

* [elasticsearch](https://registry.hub.docker.com/_/elasticsearch/)
* [logstash](https://registry.hub.docker.com/_/logstash/)
* [kibana](https://registry.hub.docker.com/_/kibana/)

* [zookeeper](https://hub.docker.com/r/confluentinc/cp-zookeeper/)
* [kafka](https://hub.docker.com/r/confluentinc/cp-kafka/)
* [kafka-rest](https://hub.docker.com/r/confluentinc/cp-kafka-rest/)
* [schema-registry](https://hub.docker.com/r/confluentinc/cp-schema-registry/)

# Requirements

## Setup

1. Install [Docker](http://docker.io).
2. Install [Docker-compose](http://docs.docker.com/compose/install/) **version >= 1.6**.
3. Clone this repository

## Increase max_map_count on your host (Linux)

You need to increase `max_map_count` on your Docker host:

```bash
$ sudo sysctl -w vm.max_map_count=262144
```

## SELinux

On distributions which have SELinux enabled out-of-the-box you will need to either re-context the files or set SELinux into Permissive mode in order for docker-elk to start properly.
For example on Redhat and CentOS, the following will apply the proper context:

```bash
.-root@centos ~
-$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

# Usage

Start the ELK stack using *docker-compose*:

```bash
$ docker-compose up
```

You can also choose to run it in background (detached mode):

```bash
$ docker-compose up -d
```

Access Kibana UI by hitting [http://localhost:5601](http://localhost:5601) with a web browser.

See: https://www.elastic.co/guide/en/kibana/current/setup.html#connect

By default, the stack exposes the following ports:
* 5000: Logstash TCP input.
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana
* 18082: Kafka REST API

*WARNING*: If you're using *boot2docker*, you must access it via the *boot2docker* IP address instead of *localhost*.

*WARNING*: If you're using *Docker Toolbox*, you must access it via the *docker-machine* IP address instead of *localhost*.

# Configuration

*NOTE*: Configuration is not dynamically reloaded, you will need to restart the stack after any change in the configuration of a component.

## Kibana configuration.

The Kibana default configuration is stored in `kibana/config/kibana.yml`.

## Logstash configuration.

The logstash configuration is stored in `logstash/config/logstash.conf`.

The folder `logstash/config` is mapped onto the container `/etc/logstash/conf.d` so you
can create more than one file in that folder if you'd like to. However, you must be aware that config files will be read from the directory in alphabetical order.

System can collects data automatically from such resources:

1. Amazon S3, configuration is stored in `logstash/config/s3-logstash.conf`, need to add credentials (every 1 minutes, default) [s3 logstash plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-s3.html).
2. Amazon RDS, configuration is stored in `logstash/config/rds-logstash.conf`, need to add credentials and JDBC driver (every 5 minutes, default) [jdbc logstash plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html).

## Specify the amount of memory used by Logstash. ##

The Logstash container use the *LS_HEAP_SIZE* environment variable to determine how much memory should be associated to the JVM heap memory (defaults to 500m).

If you want to override the default configuration, add the *LS_HEAP_SIZE* environment variable to the container in the `docker-compose.yml`:

```yml
logstash:
  build: logstash/
  command: -f /etc/logstash/conf.d/
  volumes:
    - ./logstash/config:/etc/logstash/conf.d
  ports:
    - "5000:5000"
  networks:
    - docker_elk
  depends_on:
    - elasticsearch
  environment:
    - LS_HEAP_SIZE=2048m
```

## Add Logstash plugins. ##

To add plugins to logstash you have to:

1. Add a RUN statement to the `logstash/Dockerfile` (ex. `RUN logstash-plugin install logstash-filter-json`)
2. Add the associated plugin code configuration to the `logstash/config/logstash.conf` file

## Enable a remote JMX connection to Logstash.

As for the Java heap memory, another environment variable allows to specify JAVA_OPTS used by Logstash. You'll need to specify the appropriate options to enable JMX and map the JMX port on the docker host.

Update the container in the `docker-compose.yml` to add the *LS_JAVA_OPTS* environment variable with the following content (I've mapped the JMX service on the port 18080, you can change that), do not forget to update the *-Djava.rmi.server.hostname* option with the IP address of your Docker host (replace **DOCKER_HOST_IP**):

```yml
logstash:
  build: logstash/
  command: -f /etc/logstash/conf.d/
  volumes:
    - ./logstash/config:/etc/logstash/conf.d
  ports:
    - "5000:5000"
  networks:
    - docker_elk
  depends_on:
    - elasticsearch
  environment:
    - LS_JAVA_OPTS=-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## Tune Elasticsearch configuration.

The Elasticsearch container is using the shipped configuration and it is not exposed by default.

If you want to override the default configuration, create a file `elasticsearch/config/elasticsearch.yml` and add your configuration in it.

Then, you'll need to map your configuration file inside the container in the `docker-compose.yml`. Update the elasticsearch container declaration to:

```yml
elasticsearch:
  build: elasticsearch/
  ports:
    - "9200:9200"
    - "9300:9300"
  environment:
    ES_JAVA_OPTS: "-Xms1g -Xmx1g"
  networks:
    - docker_elk
  volumes:
    - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
```

You can also specify the options you want to override directly in the command field:

```yml
elasticsearch:
  build: elasticsearch/
  command: elasticsearch -Des.network.host=_non_loopback_ -Des.cluster.name: my-cluster
  ports:
    - "9200:9200"
    - "9300:9300"
  environment:
    ES_JAVA_OPTS: "-Xms1g -Xmx1g"
  networks:
    - docker_elk
```

# Storage

## Store Elasticsearch data.

The data stored in Elasticsearch will be persisted after container reboot but not after container removal.

In order to persist Elasticsearch data even after removing the Elasticsearch container, you'll have to mount a volume on your Docker host. Update the elasticsearch container declaration to:

```yml
elasticsearch:
  build: elasticsearch/
  command: elasticsearch -Des.network.host=_non_loopback_ -Des.cluster.name: my-cluster
  ports:
    - "9200:9200"
    - "9300:9300"
  environment:
    ES_JAVA_OPTS: "-Xms1g -Xmx1g"
  networks:
    - docker_elk
  volumes:
    - /path/to/storage:/usr/share/elasticsearch/data
```

This will store elasticsearch data inside `/path/to/storage`.

# Send data to Kafka

1. First method use [PHP Kafka client](https://github.com/arnaud-lb/php-rdkafka), examples in files - `examples/kafkaConsumer.php` and `examples/kafkaProducer.php`. Available on [localhost:19092](http://localhost:19092/) after launch Docker.
2. Second method use [Kafka REST Proxy](https://github.com/confluentinc/kafka-rest). Available on [localhost:18082](http://localhost:18082/) after launch Docker.

# Analysis of log

For analysis of logs use Kibana which is available by link - [localhost:5601](http://localhost:5601/).