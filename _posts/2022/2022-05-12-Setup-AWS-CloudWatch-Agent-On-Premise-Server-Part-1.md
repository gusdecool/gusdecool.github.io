# Setup AWS CloudWatch Agent On-Premise Server — Part 1

Tutorial how to set up AWS CloudWatch agent with on-premise server so we can send logs from server outside AWS.

The second and final part is available here [PART 2](https://gusdecool.github.io/2022/06/25/Setup-AWS-CloudWatch-Agent-On-Premise-Server-Part-2-end.html)

----

## Background story why i need it
I have Kubernetes (K8s) server in Digital Ocean, currently i have difficulty on how i can persist from k8s pods as 
we know container storage is ephemeral and how i will examine the logs as it will require another tool to read it easily.

For temporary solution, i have micro web service as the central for log storage called “Tracing” and my applications 
will send the logs to Tracing via HTTP request instantly for each log. Then soon i realized this method is too expensive
and slow as each HTTP call cost time.

Thus why i need something faster yet minimum maintenance. Since i got quite familiar AWS CloudWatch, i would like to 
use it as my central log, but since my K8s server is outside of AWS, i need to find out how to do it. Luckly AWS have
prepared AWS CloudWatch agent which it will act as an agent (hence its name) to send the logs to AWS.

-----

## Options to install CloudWatch Agent
There are 2 options i knew how to install CloudWatch agent with my familiarity with Docker & K8s.

Install it as binary/service manually in my app operating system in each container. I tried this then found how 
complicated it’s and no luck to make it works. Then i realized if i do it this way, i will need to manage the 
CloudWatch agent for each of my containers which will cost me another overhead. I drop this idea.
Install it as container using AWS provided Docker image as in this https://hub.docker.com/r/amazon/cloudwatch-agent. 

The problem with this image is it minimum documentation focused on using this Docker Image 😢
So on this part 1, i want to describe how i successfully do it with container. Since i successfully did it by run 
trial and error reading each container error message and search on Google to make sense of it. I hope this post will 
help you and save your time.

-----

## Prerequisites
Before you can start following this tutorial, below is some prerequisites that you must have:

1. AWS account (obviously)
2. AWS IAM account with access to create log group and stream. You will need to have it access key and secret. Let me 
know if you need my help to describe IAM access specs. 
3. Knowledge how to use Docker.

-----

## Docker Compose
In this tutorial, i will use Docker Compose instead of pure Docker CLI. Since i feel it’s easier for me to record the change i made and git commit for history.

This is the config of my `docker-composer.yml`

```yml
version: "3.8"
services:
  agent:
    image: amazon/cloudwatch-agent
    volumes:
      - ./config/log-collect.json:/opt/aws/amazon-cloudwatch-agent/bin/default_linux_config.json
      - ./aws:/root/.aws
      - ./log:/log
      - ./etc:/opt/aws/amazon-cloudwatch-agent/etc
```

Do not run `docker-compose up` yet as still didn’t have the sync volume files which it the important file that i will explain below.

-----

## CloudWatch agent config
CloudWatch agent config is the configuration that describe how the agent will collect the log from the container and 
send it to AWS. In my context it represented in as file `./config/log-collect.json` then synced to the container.

Below is the content of that file

```json
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
            "log_group_name": "container-aws-logs-test",
            "log_stream_name": "{hostname}",
            "timestamp_format" :"[%Y-%m-%dT%H:%M:%S.%f%z]",
            "retention_in_days": 30
          }
        ]
      }
    }
  }
}
```

You can read the config structure docs at https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html

I will explain in detail the keypoints of my config

1. “`metrics_collection_interval`” describe in what every seconds the agent will check the log file and send to AWS. 
2. 60 seconds is the default. Do not set it too soon if not needed as it will increase the charge of AWS server.
“`run_as_user`” i set it as root. Feel free to set it as any user other than root if you want to have it extra secure.
3. I did not collecting “`log_stream_name`” as it run in stand alone container, so it didn’t make sense to capture 
the metric of AWS CloudWatch agent container itself. And since I’m using K8s auto scaling, this is not an important issue
for me to monitor for now.
4. “`collect_list`” describe which files the agent will watch and send to AWS. The interesting part in there, this is an 
array of object. Which mean we possibly can describe multiple files to watch then only need to have 1 agent or 1 
container to watch all my pods for saving computing resource purpose.
5. “`file_path`” this describe which file the agent will read.
6. “`log_group_name`” & “`log_stream_name`” describe to which group and stream this log will belong in AWS CloudWatch.
7. “`timestamp_format`” describe your logs timestamp. If it match, CloudWatch will use your log timestamp, and if not,
it will use current timestamp. I found difficulty when configuring and testing which will i describe how i solve it below.
8. “`retention_in_days`” describe how log the log will persist. 30 days is enough for testing.

-----

## AWS Credentials
AWS credentials will be used to authenticate to be able to send log to your aws account. It represented as folder 
`./aws`, inside that folder i have 2 AWS standard credentials files like below:

`./aws/config`

```ini
[profile AmazonCloudWatchAgent]
output = text
region = ap-southeast-1
```

it’s important to name it as “AmazonCloudWatchAgent” as the container designed to use this profile. Change the region 
into your AWS region or any aws region you want to record the log.

`./aws/credentials`

```ini
[AmazonCloudWatchAgent]
aws_access_key_id = IAM_ID
aws_secret_access_key = IAM_KEY
```

Again, it’s important to name the profile `AmazonCloudWatchAgent`.

And that’s it for aws credentials.

-----

## Log file
This is not required in production but good to do to test if the container run in development before we go to production
and help you understand the concept of how the agent monitor the file.

`./log/app.log` it contains logs file from Symfony application with a modified timestamp (as i mentioned above, 
I got difficulty with timestamp). Note the filename is “app.log” which match with the JSON config from file 
`./config/log-collect.json` that i configured above.

My content of log directory is a below

```txt
[2022-05-14T17:32:19.826204+0000] app.INFO: budi test {"budi":"foo"}
```

The content is pretty simple.

-----

## Time to test
With all that configured, now you’re ready to start the docker with command “docker compose up”. After it run, examine 
the output in the terminal, then check in your AWS account CloudWatch group “container-aws-logs-test” to see if it
successfully delivered.

If you're facing any error, it most likely due to IAM account used didn’t have access to create log.

Now try to add any new line in file `./log/app.log”, wait 60 seconds and see it it delivered.

Now you understand the concept how the agent works. **CONGRATULATION!**

-----

## Bonus: Debugging
As i mentioned earlier, i have difficulty on how to setup the timestamp and there is no way for me to SSH into the
container as it didn’t have “bash” nor “sh” so i unable to “docker compose exec sh” into container. And there is now 
way for me to install the “bash” as the container not even have “apt-get” command, i guess it’s because the image is 
created using container_linux:go. I don’t know how AWS did it or build the image, it’s something that i need to learn
in the future.

After Google-fu for a moment and examining the terminal output, i learnt that the Agent config is located at 
“/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.toml” inside the container. And since i’m unable to SSH
into the container to view the file, i got idea to sync that file into my local computer. Hence you see the config sync
volume of

```yml
volumes:
- ....
- ./etc:/opt/aws/amazon-cloudwatch-agent/etc
```

now if you read the file “./etc/amazon-cloudwatch-agent.toml” in your local file, i found this interesting lines

```toml
timestamp_layout = "[2006-01-02T15:04:05..000-0700]"
timestamp_regex = "(\\[\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.(\\d{1,9})[\\+-]\\d{4}\\])"
```

Aha! this is the timestamp config used by the Agent. But my problem not end there as then i found out the 
“timestamp_layout” example as not accurate 😢. Notice the timestamp between second and microseconds/milliseconds 
“05..000” there are 2 dots (..). This is inaccurate as if looks from the regex it only check for 1 dot.

So my suggest is to stick with test it with regex, you can test your log file input with regex validation tool 
at https://regex101.com/.

-----

## Next part
As this proof of concept success. My next step will be how to install this into K8s. Since this run as container, I 
have two options to do it.

1. Have CloudWatch Agent as single independent pod then all my apps will have shared storage to this Agent pod. So the 
Agent pod will read all the storage files provided by all app pods. The pros: saving computing resources. The cons: I
will need to update the config of this Agent pod whenever i add new app and this potentially will have lot of configuration.
2. Have CloudWatch Agent installed as container in the same pod with the app. So each app have it own Agent. The pros:
the agent have small config and the config will also simpler. The cons: every app will have it own agent and when 
horizontal scaling happened, the agent will also replicated, my concern is with computing resource. But i guess it will
quite small for starter.
3. At the moment i lean to option 2. Still not sure, but i will try that first and will let you know how it goes in 
next post. Once i did it, i will update this post to link to the next post.

Stay tune…

Thank you for reading ✌️