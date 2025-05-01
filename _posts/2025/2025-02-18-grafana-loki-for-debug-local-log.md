# Grafana Loki for Debug Local Log
> My experience setup and using Grafana Loki for debugging local log.

## The issue
For years, I've been using AWS CloudWatch & Log Insight to debug my application on AWS infrastructure, it's works great. 
But when I'm developing locally, I'm still using Jetbrains IDE to read the log manually and inspect it using editor. 

This is not an ideal situation because after several minutes of the development, the log file is getting bigger and bigger, and it's hard to find the log that I need.
Especially when the editor did syntax processing on the lo file, the process will be much slower. This made me need to regularly delete the log file to make it faster.

## What I want to achieve
I want to have a tool that can inspect log like what CloudWatch do but locally. 
I could send the log from local file to AWS CloudWatch using external provider method (like in my older post), but it doesn't feel efficient.

Thus begin the research to find the suitable tools. 
Ideally I want the tools to be easy to set up, replicate and remove. 
With this specification, I already narrowed my options that it must support or use Docker (even if it didn't have Docker image, I can create one myself).

It should support reading log from multiple applications. 
I have multiple apps in development, I didn't want each app will have their own tools to inspect log, as it will make the app development heavier and too much to manage. 

## Grafana Loki
After searching for a while, I found Grafana Loki intriguing as it's open source and support multiple log sources.
It took me a while to understand which services to use since Grafana has a lot of services and need to understand their terminology.
Luckily, Grafana already provide the docker image. So what I need to do is just to configure the docker-compose file and run it.

Below is my docker-compose setup for Grafana Loki while I'm explaining what some line does and describe in simpler terminology for each Grafana service used:

```yaml
# docker-compose.yml
networks:
  loki:

services:
  # UI to view log
  grafana:
    # This image is just a web interface to used all the grafana service, but the service itself is not here. 
    # We need to use other image to run backend of Grafana service. e.g: Loki
    image: grafana/grafana:11.5.1 
    ports:
      # the port where I will access the Grafana UI
      - "3000:3000"
    networks:
      - loki
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=true # since this is a local usage, I don't need authentication, so I can quickly open Grafana.
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_FEATURE_TOGGLES_ENABLE=alertingSimplifiedRouting,alertingQueryAndExpressionsStepMode
    entrypoint:
      - sh
      - -euc
      # config the Grafana data source for Loki
      # notice, it's using http://loki:3100, this is because the service name is loki, and it will be resolved to the IP address of the loki service.
      - |
        mkdir -p /etc/grafana/provisioning/datasources
        cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
        apiVersion: 1
        datasources:
        - name: Loki
          type: loki
          access: proxy 
          orgId: 1
          url: http://loki:3100
          basicAuth: false
          isDefault: true
          version: 1
          editable: false
        EOF
        /run.sh

  # the agent to collect the logs
  promtail:
    # Promtail is a service agent that will read the raw log file and then sent it Loki.
    # Note: Promtail is deprecated, but if it works for me, that's fine.
    # good practice to also define the exact image version, in case the newer version has breaking changes. Avoid surprise!
    image: grafana/promtail:3.4 
    volumes:
      # the config file for promtail, I will provide below
      - ./promtail-config.yaml:/etc/promtail/config.yml 
        
      # I have multiple apps which is equal to multiple logs files but I only 1 instance of log inspector to manage all.
      # This is the trick to make it works, I mount the log file from multiple apps to the same promtail container.
      - ~/app-1-laravel/storage/logs:/var/log/app-1-laravel
      - ~/app-2-symfony/var/log:/var/log/app-2-symfony
    command: -config.file=/etc/promtail/config.yml
    networks:
      - loki
  
  # the agent that query the log
  loki:
    # Loki is a service that will store the log and provide the ability to query the log (searching, filtering, etc).
    image: grafana/loki:3.4
    ports:
      # the port I open, identical with grafana service config above.
      - "3100:3100"
    # the config file for Loki, I will provide below
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki
```

## Promtail config
Below is the `promtail-config.yaml` I mentioned above that I use to configure the promtail agent.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push # note this is the Loki service URL, it will be resolved to the IP address of the loki service.

scrape_configs:
  - job_name: app-1-laravel
    static_configs:
      - targets:
          - localhost
        labels:
          job: app-1-laravel # label to use to filter later in UI
          __path__: /var/log/app-1-laravel/*log # get all the file with extension log in this directory
    # by default, Loki will try to determine the log fields but from my experience it's only able to determine the datetime & log level automatically.
    # so I need to define the log fields manually. To do that, I will need to use pipeline_stages below. I will give example of the log context and explain the regex.
    pipeline_stages:
      - regex:
          # regex expression to parse the log line into a captured named group.
          expression: '\[(?P<log_date>[^]]+)\] (?P<log_app>[^:]+): (?P<log_ip>[^ ]+) "(?P<log_group>[^"]*)" "(?P<log_supplier>[^"]*)" "(?P<log_message>[^"]*)" (?P<log_context>{.*})'
      - labels:
          log_date: log_date
          log_app: log_app
          log_ip: log_ip
          log_group: log_group
          log_supplier: log_supplier
          log_message: log_message
          log_context: log_context
  - job_name: app-2-symfony
    static_configs:
      - targets:
          - localhost
        labels:
          job: app-2-symfony
          __path__: /var/log/app-2-symfony/*log
```

Example of my log single line for `app-1-laravel` is like below.

> [2025-02-18 02:24:14] local.INFO: 192.168.65.1 "StripePaymentIntentCreateHttpRequest" "EC" "http request to create payment intent" 
> {"sessionId":"f72f438b-494d-4a0d-9d74-f93cec2e492c","payload":{"public_key":"pk_test_redacted","params":{"amount":1000,"currency":"aud"}}} []

We have info of
1. datetime when the log is created
2. local.INFO is the app and log level
3. 192.168.65.1 is the user IP address. Obviously, this is the local IP address, but in production, it will be the user IP address.
4. "StripePaymentIntentCreateHttpRequest" is the group of the log or what it's doing.
5. "EC" is the supplier ID. Sometimes it can be empty, since not all process have info of the supplier
6. "http request to create payment intent" is the descriptive message of the log
7. {"sessionId":"f72f438b-494d-4a0d-9d74-f93cec2e492c", .....} values are the log context. There is sessionId in there is a trick I used to monitor what is happening in a single HTTP request,
as in Production with millions of HTTP requests, that will be hard to do.

For sessionId, if you interested in how I built it. Let me know. I will create another post to explain how I built it.

## Loki config
Below is the `local-config.yaml` I mentioned above that I use to configure the Loki service.

```yaml
server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093
```

There is nothing special in the config, I just use the default config that Grafana provided. 
(I forgot from where I copied it, but it's from the official documentation, I will update the post if find it).

## Conclusion
Once this done, I just need to run `docker-compose up -d` and I can access the Grafana UI at `http://localhost:3000` and start inspecting the log.
This process can also replicate to my team member without any hassle as they just need to copy the repo and start the docker compose.

-----
If you found my blog post insightful and valuable, you can support my work with a voluntary contribution. 
Your support helps sustain independent writing, research, and the continued sharing of high-quality content.

**Why Donate?**  
- Encourages the creation of more in-depth, well-researched content.
- Helps cover costs like hosting, tools, and time spent on writing.
- Supports independent writing without paywalls or intrusive ads.

**How It Works:**  
- This is a voluntary contribution with a minimum of $3—you can choose any amount.
- 100% of your support goes toward improving and expanding my content.
- Your contribution is greatly appreciated.

**Have a Topic in Mind?**  
If there's a specific topic you'd like me to cover, feel free to reach out! You can email me at [budi.arsana@bungamata.com](mailto:budi.arsana@bungamata.com), and I'll consider it for future content.

Support me at [https://budiarsana.gumroad.com/coffee](https://budiarsana.gumroad.com/coffee)
