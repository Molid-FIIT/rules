# Kompromitacia klucov
- Chirpstack pouziva pre ukladanie citlivych udajov REDIS
- Docker-compose.yml bol upraveny tak aby redis pocuval na lokalnom porte 6379 chirpstack serveru
- Bol stiahnuty, vytvoreny a presunuty na cestu subor `redis-cli`, postup je nizsie uvedeny
- Je vytvorena screen session `redis-logs`, v ktorej bezi jediny skript `redis-logs.sh`, obsah skriptu je uvedeny nizsie
- Tento skript parsuje udaje z `redis-cli monitor` do JSON, ktory je nasledne ukladany na ceste `/data/redis.log`
- Pomocou filebeat sa presuva obsah `redis.log` na logstash pipeline port 5702
- Konfiguracia pipeline je uvedena nizsie
- Pre ELK SIEM je vytvorene pravidlo uvedene nizsie
- Pravidlo spociva v tom, ze ak sa na redis pripoji cudzia IP adresa, tak sa vygeneruje alert, lebo automaticky dana IP-cka moze mat pristup k privatnym klucom

## Upraveny docker-compose.yml
- Dolezita zmena je port 6379, ktory meni pocuvanie len na localhost a vobec umoznuje pocuvat na danom porte hlavnemu stroju:
    ```yml
        redis:
            image: redis:7-alpine
            restart: unless-stopped
            volumes:
            - redisdata:/data
            ports:
            - 127.0.0.1:6379:6379
    ```
- nahradte `PASSWORD` nizsie za platne hesla
```yml
version: "3"

services:
  chirpstack:
    image: chirpstack/chirpstack:4.2.0
    command: -c /etc/chirpstack
    restart: unless-stopped
    volumes:
      - ./configuration/chirpstack:/etc/chirpstack
      - ./lorawan-devices:/opt/lorawan-devices
    depends_on:
      - postgres
      - mosquitto
      - redis
    environment:
      - MQTT_BROKER_HOST=PASSWORD
      - REDIS_HOST=PASSWORD
      - POSTGRESQL_HOST=PASSWORD
    ports:
      - 8001:8080

  chirpstack-gateway-bridge-eu868:
    image: chirpstack/chirpstack-gateway-bridge:4.0.6
    restart: unless-stopped
    ports:
      - 8005:1700/udp
    volumes:
      - ./configuration/chirpstack-gateway-bridge:/etc/chirpstack-gateway-bridge
    depends_on: 
      - mosquitto

  chirpstack-rest-api:
    image: chirpstack/chirpstack-rest-api:4.2.0
    restart: unless-stopped
    command: --server chirpstack:8080 --bind 0.0.0.0:8090 --insecure
    ports:
      - 8090:8090
    depends_on:
      - chirpstack

  postgres:
    image: postgres:14-alpine
    restart: unless-stopped
    volumes:
      - ./configuration/postgresql/initdb:/docker-entrypoint-initdb.d
      - postgresqldata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=PASSWORD

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data
    ports:
      - 127.0.0.1:6379:6379

  mosquitto:
    image: eclipse-mosquitto:2
    restart: unless-stopped
    ports:
      - 1883:1883
    volumes: 
      - ./configuration/mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf

volumes:
  postgresqldata:
  redisdata:
```

## Kompilacia Redis
- ZDROJ: https://codewithhugo.com/install-just-redis-cli-on-ubuntu-debian-jessie/
- You’ll need libjemalloc1 libjemalloc-dev gcc make most of which should already be installed. We’re building from source… which takes about a minute on the CircleCI containers (so I would expect less everywhere else), which is fine. Credit: DevOps Zone, install redis-cli without installing server. I shamelessly took the snippet from there, because hey, it works.
```sh
cd /tmp
wget http://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
cp src/redis-cli /usr/local/bin/
chmod 755 /usr/local/bin/redis-cli
```

## Skript redis-logs.sh
- pouzite `screen -S redis-logs` a v screen session nasledne len pustite skript `./redis-logs.sh`
- ma to vela nevyhod avsak pre potreby monitorovania je to dostatocne
- chyba rotovanie logov a automaticke spustenie po starte stroja
```sh
#!/bin/bash

redis-cli monitor | sed 's/\[//g' | sed 's/\]//g' | sed 's/:/ /' | awk '{print "{\"timestamp\":\""$1"\",\"ip\":\""$3"\",\"port\":\""$4"\",\"action\":"$5"}"}' > /data/redis.log 2>/dev/null
```

## Logstash pipeline
- /etc/logstash/conf.d/redis-pipeline.conf
```conf
input {
  beats {
    port => "5702"
  }
}

filter {
  json {
    source => "message"
    target => "data"
    remove_field => ["message"]
  }
}

output {
  elasticsearch {
    hosts => "https://127.0.0.1:9200"
    ssl_certificate_verification => false
    index => "redis-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "L=kPpanmLU8+vE8sAC8O"
  }
}
```
### Ostatne logstash pipeline
- /etc/logstash/pipelines.yml
  ```yml
    # This file is where you define your pipelines. You can define multiple.
    # For more information on multiple pipelines, see the documentation:
    #   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

    - pipeline.id: chirpstack
    path.config: "/etc/logstash/conf.d/chirpstack-pipeline.conf"
    - pipeline.id: gateways
    path.config: "/etc/logstash/conf.d/gateways-pipeline.conf"
    - pipeline.id: redis
    path.config: "/etc/logstash/conf.d/redis-pipeline.conf"
  ```
- /etc/logstash/conf.d/chirpstack-pipeline.conf
```conf
input {
  http {
    port => 5700
    response_headers => {
      "Access-Control-Allow-Origin" => "*"
      "Content-Type" => "application/json"
      "Access-Control-Allow-Headers" => "Origin, X-Requested-With, Content-Type, Accept"
    }
  }
}

filter {
  ruby {
    code => 'event.set("data-json", Base64.decode64(event.get("data")))'
  }
  json {
    source => "data-json"
    target => "data-unpacked"
  }
  mutate {
    convert => {
      "[data-unpacked][temp]" => "integer"
    }
  }
} 

output {
  elasticsearch {
    hosts => "https://127.0.0.1:9200"
      ssl_certificate_verification => false
    index => "chirpstack-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "L=kPpanmLU8+vE8sAC8O"
  }
}
```
- /etc/logstash/conf.d/gateways-pipeline.conf 
```conf
input {
  beats {
    port => "5701"
  }
}

filter {
  json {
    source => "message"
    target => "data"
    remove_field => ["message"]
  }
  mutate {
    convert => {
      "[data][lgwm]" => "string"
    }
  }
}   

output {
  elasticsearch {
    hosts => "https://127.0.0.1:9200"
    ssl_certificate_verification => false
    index => "lora-gateway-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "L=kPpanmLU8+vE8sAC8O"
  }
}
```

## Rule pre ELK SIEM
```json
{"id":"5321df10-dc70-11ed-a1ab-5dee06af8c3c","updated_at":"2023-04-16T19:53:33.921Z","updated_by":"elastic","created_at":"2023-04-16T16:04:11.659Z","created_by":"elastic","name":"KEYS","tags":["private-keys"],"interval":"1m","enabled":true,"description":"Watch for Private key reads from Redis database on chirpstack","risk_score":21,"severity":"low","license":"","output_index":"","meta":{"from":"30s","kibana_siem_app_url":"http://10.0.130.124:5601/app/security"},"author":[],"false_positives":[],"from":"now-90s","rule_id":"d203f28c-c568-4da7-bd1e-a36350d23690","max_signals":100,"risk_score_mapping":[],"severity_mapping":[],"threat":[],"to":"now","references":[],"version":3,"exceptions_list":[],"immutable":false,"related_integrations":[],"required_fields":[],"setup":"","type":"new_terms","query":"*","new_terms_fields":["data.ip.keyword"],"history_window_start":"now-1d","filters":[],"language":"kuery","data_view_id":"90ef18c3-04a0-412f-b19f-f51e27273074","throttle":"rule","actions":[{"group":"default","id":"6d4d7dc0-cd31-11ed-a1ab-5dee06af8c3c","params":{"documents":[{"name":"private_key"}]},"action_type_id":".index"}]}
{"exported_count":1,"exported_rules_count":1,"missing_rules":[],"missing_rules_count":0,"exported_exception_list_count":0,"exported_exception_list_item_count":0,"missing_exception_list_item_count":0,"missing_exception_list_items":[],"missing_exception_lists":[],"missing_exception_lists_count":0}
```

## Filebeat konfiguracia
- pozor ziaden `modules.d` nemusi byt aktivny, t.j. vsetky subory na ceste `/etc/filebeat/modules.d/` musia mat priponu `.disabled`.
```yml
###################### Filebeat Configuration Example #########################

# This file is an example configuration file highlighting only the most common
# options. The filebeat.reference.yml file from the same directory contains all the
# supported options with more comments. You can use it as a reference.
#
# You can find the full configuration reference here:
# https://www.elastic.co/guide/en/beats/filebeat/index.html

# For more available modules and options, please see the filebeat.reference.yml sample
# configuration file.

# ============================== Filebeat inputs ===============================

filebeat.inputs:

# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.
# Below are the input specific configurations.

# filestream is an input for collecting log messages from files.
- type: filestream

  # Unique ID among all inputs, an ID is required.
  id: molid-redis

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /data/redis.log
    #- c:\programdata\elasticsearch\logs\*

  # Exclude lines. A list of regular expressions to match. It drops the lines that are
  # matching any regular expression from the list.
  # Line filtering happens after the parsers pipeline. If you would like to filter lines
  # before parsers, use include_message parser.
  #exclude_lines: ['^DBG']

  # Include lines. A list of regular expressions to match. It exports the lines that are
  # matching any regular expression from the list.
  # Line filtering happens after the parsers pipeline. If you would like to filter lines
  # before parsers, use include_message parser.
  #include_lines: ['^ERR', '^WARN']

  # Exclude files. A list of regular expressions to match. Filebeat drops the files that
  # are matching any regular expression from the list. By default, no files are dropped.
  #prospector.scanner.exclude_files: ['.gz$']

  # Optional additional fields. These fields can be freely picked
  # to add additional information to the crawled log files for filtering
  #fields:
  #  level: debug
  #  review: 1

# ============================== Filebeat modules ==============================

filebeat.config.modules:
  # Glob pattern for configuration loading
  path: ${path.config}/modules.d/*.yml

  # Set to true to enable config reloading
  reload.enabled: false

  # Period on which files under path should be checked for changes
  #reload.period: 10s

# ======================= Elasticsearch template setting =======================

setup.template.settings:
  index.number_of_shards: 1
  #index.codec: best_compression
  #_source.enabled: false


# ================================== General ===================================

# The name of the shipper that publishes the network data. It can be used to group
# all the transactions sent by a single shipper in the web interface.
#name:

# The tags of the shipper are included in their own field with each
# transaction published.
#tags: ["service-X", "web-tier"]

# Optional fields that you can specify to add additional information to the
# output.
#fields:
#  env: staging

# ================================= Dashboards =================================
# These settings control loading the sample dashboards to the Kibana index. Loading
# the dashboards is disabled by default and can be enabled either by setting the
# options here or by using the `setup` command.
#setup.dashboards.enabled: false

# The URL from where to download the dashboards archive. By default this URL
# has a value which is computed based on the Beat name and version. For released
# versions, this URL points to the dashboard archive on the artifacts.elastic.co
# website.
#setup.dashboards.url:

# =================================== Kibana ===================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"

  # Kibana Space ID
  # ID of the Kibana Space into which the dashboards should be loaded. By default,
  # the Default Space will be used.
  #space.id:

# =============================== Elastic Cloud ================================

# These settings simplify using Filebeat with the Elastic Cloud (https://cloud.elastic.co/).

# The cloud.id setting overwrites the `output.elasticsearch.hosts` and
# `setup.kibana.host` options.
# You can find the `cloud.id` in the Elastic Cloud web UI.
#cloud.id:

# The cloud.auth setting overwrites the `output.elasticsearch.username` and
# `output.elasticsearch.password` settings. The format is `<user>:<pass>`.
#cloud.auth:

# ================================== Outputs ===================================

# Configure what output to use when sending the data collected by the beat.

# ---------------------------- Elasticsearch Output ----------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["localhost:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"

# ------------------------------ Logstash Output -------------------------------
output.logstash:
  # The Logstash hosts
  hosts: ["193.87.2.13:15702"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

# ================================= Processors =================================
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

# ================================== Logging ===================================

# Sets log level. The default log level is info.
# Available log levels are: error, warning, info, debug
#logging.level: debug

# At debug level, you can selectively enable logging only for some components.
# To enable all selectors use ["*"]. Examples of other selectors are "beat",
# "publisher", "service".
#logging.selectors: ["*"]

# ============================= X-Pack Monitoring ==============================
# Filebeat can export internal metrics to a central Elasticsearch monitoring
# cluster.  This requires xpack monitoring to be enabled in Elasticsearch.  The
# reporting is disabled by default.

# Set to true to enable the monitoring reporter.
#monitoring.enabled: false

# Sets the UUID of the Elasticsearch cluster under which monitoring data for this
# Filebeat instance will appear in the Stack Monitoring UI. If output.elasticsearch
# is enabled, the UUID is derived from the Elasticsearch cluster referenced by output.elasticsearch.
#monitoring.cluster_uuid:

# Uncomment to send the metrics to Elasticsearch. Most settings from the
# Elasticsearch output are accepted here as well.
# Note that the settings should point to your Elasticsearch *monitoring* cluster.
# Any setting that is not set is automatically inherited from the Elasticsearch
# output configuration, so if you have the Elasticsearch output configured such
# that it is pointing to your Elasticsearch monitoring cluster, you can simply
# uncomment the following line.
#monitoring.elasticsearch:

# ============================== Instrumentation ===============================

# Instrumentation support for the filebeat.
#instrumentation:
    # Set to true to enable instrumentation of filebeat.
    #enabled: false

    # Environment in which filebeat is running on (eg: staging, production, etc.)
    #environment: ""

    # APM Server hosts to report instrumentation results to.
    #hosts:
    #  - http://localhost:8200

    # API Key for the APM Server(s).
    # If api_key is set then secret_token will be ignored.
    #api_key:

    # Secret token for the APM Server(s).
    #secret_token:


# ================================= Migration ==================================

# This allows to enable 6.7 migration aliases
#migration.6_to_7.enabled: true

```