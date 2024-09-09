# Setup AWS CloudWatch Agent On-Premise Server — Part 2 [END]

Tutorial how to add AWS CloudWatch agent in Kubernetes server.

Previous post at [PART 1](https://gusdecool.github.io/2022/05/12/Setup-AWS-CloudWatch-Agent-On-Premise-Server-Part-1.html)

On part 1 we have successfully tested AWS CloudWatch agent (I will short it as CWA in next mentioned) as a container. 
Now in this post I will share how I successfully installed in my K8s server.

I will share you my full K8s declarative config first, then I will explain each line that related and important. 
In this example I using my Symfony application as the app that will write log, you can change it into any of your app,
as long as your app write a log file, we can use it and make CWA send log event to AWS.

## My K8s full config

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-prod
  template:
    metadata:
      labels:
        app: api-prod
    spec:
      containers:
        - image: your-application-image
          name: web
          envFrom:
            - configMapRef:
                name: api-prod-env
          volumeMounts:
            - mountPath: /app/var/log
              name: app-log
        - image: amazon/cloudwatch-agent:1.247350.0b251814
          name: agent
          volumeMounts:
            - mountPath: /etc/cwagentconfig
              name: agent-config
              readOnly: true
            - mountPath: /log
              readOnly: true
              name: app-log
            - mountPath: /root/.aws
              name: aws-cred
              readOnly: true
      volumes:
        - name: app-log
          emptyDir: { }
        - name: agent-config
          configMap:
            name: api-prod-cwagent
            items:
              - key: cwagentconfig
                path: cwagentconfig
        - name: aws-cred
          configMap:
            name: aws-cred
      terminationGracePeriodSeconds: 60
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-prod-cwagent
data:
  cwagentconfig: |
    {
      "agent": {
        "metrics_collection_interval": 60,
        "run_as_user": "root"
      },
      "logs": {
        "logs_collected": {
          "files": {
            "collect_list": [
              {
                "file_path": "/log/app.log",
                "log_group_name": "api-prod",
                "log_stream_name": "api-prod-{hostname}",
                "timestamp_format" :"[%Y-%m-%dT%H:%M:%S.%f%z]",
                "retention_in_days": 365
              }
            ]
          }
        }
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-cred
data:
  config: |
    [profile AmazonCloudWatchAgent]
    output = text
    region = ap-southeast-1
  credentials: |
    [AmazonCloudWatchAgent]
    aws_access_key_id = XXX-XXX
    aws_secret_access_key = XXX-XXX
```

## Deployment config explanation
### Container config
If you follow my previous post, I decided to config CWA installed on each pod. This may be not optimal as we could end up 
having too many CWA agent, but this should work for now and I prefer to keep it simple for starter and proof of concept.

I will explain each relevant config by it excerpt.

```yaml
        - image: your-application-image
          name: web
          envFrom:
            - configMapRef:
                name: api-prod-env
          volumeMounts:
            - mountPath: /app/var/log
              name: app-log```
```

This if my core container app, this image contain my full application. It read envFrom K8s configMapRef, I will not 
explain in detail for each as each app is different and not relevant with CWA installation.

The interesting part is “volumeMounts”, my core container app write log to file “/app/var/log/app.log” which this 
file is then synced with other container in CWA, so CWA have capabilities to read the log file from different 
container (decoupled concept).

```yaml
        - image: amazon/cloudwatch-agent:1.247350.0b251814
          name: agent
          volumeMounts:
            - mountPath: /etc/cwagentconfig
              name: agent-config
              readOnly: true
            - mountPath: /log
              readOnly: true
              name: app-log
            - mountPath: /root/.aws
              name: aws-cred
              readOnly: true
```

Above is the CWA container, there are only “volumeMounts” which act ac the container config. I will explain each in detail below.

```yaml
            - mountPath: /etc/cwagentconfig
              name: agent-config
              readOnly: true
```

This is the CWA core config. It configure how the CWA should act like which file to read. If you read my previous, 
this is the “cwagentconfig.cnf” file.

```yaml
            - mountPath: /log
              readOnly: true
              name: app-log
```

Above is the volume of the log file, which it synced the core app container. Thing to note here, if you config the CWA 
to delete the log lines once it send to AWS CloudWatch, you may want to set “readOnly: false”.

```yaml
            - mountPath: /root/.aws
              name: aws-cred
              readOnly: true
```

Above is your AWS CWA agent credentials where the container will read it from file. It contains the “aws_access_key_id”
& “aws_secret_access_key”.

### Deployment volume config
```yaml
        volumes:
        - name: app-log
          emptyDir: { }
        - name: agent-config
          configMap:
            name: api-prod-cwagent
            items:
              - key: cwagentconfig
                path: cwagentconfig
        - name: aws-cred
          configMap:
            name: aws-cred
```

Above is the deployment volume config. I will explain each below

```yaml
- name: app-log
  emptyDir: { }
```

Above the is “app-log”, i decided to go with “emptyDir” volume because the content of this volume is not important to 
keep once the log sent to AWS CloudWatch.

```yaml
        - name: agent-config
          configMap:
            name: api-prod-cwagent
            items:
              - key: cwagentconfig
                path: cwagentconfig
```

Above is CWA config, since K8s support using “configMap” as “file”. I decided to use configMap to keep all configs in 
a single file rather than reference it to external file. Just to keep thing simple and have everything in a single 
deployment file.

```yaml
        - name: aws-cred
          configMap:
            name: aws-cred
```

Above is the AWS credentials file. Similar like “agent-config”, this is the “configMap” as “file”

### Pod termination timing
```yaml
terminationGracePeriodSeconds: 60
```

I set the “terminationGracePeriodSeconds: 60” to follow the CWA “metrics_collection_interval: 60” interval, so when the
pod replaced with new deployment, it will have grace period for 60 seconds to allow CWA send the log the AWS Cloudwatch,
you may want to increase this value a little bit, e.g: by 10 seconds to be safe.

### ConfigMap “api-prod-cwagent”
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-prod-cwagent
data:
  cwagentconfig: |
    {
      "agent": {
        "metrics_collection_interval": 60,
        "run_as_user": "root"
      },
      "logs": {
        "logs_collected": {
          "files": {
            "collect_list": [
              {
                "file_path": "/log/app.log",
                "log_group_name": "api-prod",
                "log_stream_name": "api-prod-{hostname}",
                "timestamp_format" :"[%Y-%m-%dT%H:%M:%S.%f%z]",
                "retention_in_days": 365
              }
            ]
          }
        }
      }
    }
```

Above is the ConfigMap of the CWA. It's basically using K8s capability to create a file from a config map. The config for CWA
has been explained in post 1, I will not explain again here.

### ConfigMap “aws-cred”
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-cred
data:
  config: |
    [profile AmazonCloudWatchAgent]
    output = text
    region = ap-southeast-1
  credentials: |
    [AmazonCloudWatchAgent]
    aws_access_key_id = XXX-XXX
    aws_secret_access_key = XXX-XXX
```

Above are the AWS credentials, you need to replace the value of access key id and secret access key with yours. 
Then K8s will create a file from this config map.

-----

That’s pretty much what I did to successfully setup CWA in K8s. If you have any questions or other strategies how to 
set in K8s, please let me know in comment.

And last, I apologize for the delay writing this post final part. I actually already resolved it a week after the first 
post, but only recently got the time to write this post.

Thank you.