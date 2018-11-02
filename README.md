# Athena Syslogger

Use Fluentd to send syslogs to Athena for great querying.

## Prerequisites

One of the following depending on usage method.

- Ruby 2.x
- Docker
- AWS

## Overview

Goal: A simple-to-setup syslog endpoint that can drop logs into S3
so that they can be easily queried with Amazon Athena.

Fluentd has plugins for both syslog and S3 and can reliably deliver logs in JSON format.

## Usage

There are three options for using this system, each with varying levels of setup and configuration.

1. [Ruby-based installation](#ruby-installation) - Install and run Fluentd yourself, good for testing.
2. [Docker](#docker) - Run fluentd locally or on your server using a Docker container.
3. [AWS Fargate](#aws-fargate) - Use containers in the cloud to manage everything!

### Configuration

For any of these approaches, you must first configure Fluentd to your liking.

An example config file is in [s3-fluentd/fluent.conf](s3-fluentd/fluent.conf).

Specifically, you'll need to configure `s3_bucket` and `s3_region`.

If you _do not_ have AWS credentials already configured, you'll also need to
add `aws_key_id` and `aws_sec_key`. The methods below assume that you can already use aws cli commands to access S3 or that you will be passing in access/secret keys as needed. These credentials need access to write to the `s3_bucket` provided above.

### Ruby installation

1. Install fluentd and the s3 output plugin. 

    Full install instructions can be found here: https://docs.fluentd.org/v1.0/articles/install-by-gem

    ```shell
    gem install fluentd fluent-plugin-s3 --no-ri --no-rdoc
    ```

2. Start up fluentd and start logging!

    ```shell
    fluentd -c s3-fluentd/fluent.conf
    ```

By default, fluentd will listen for syslog connections on UDP port 5140.

To test, you can use netcat.

```shell
echo '<13>'$(date -u +"%b %d %T")' 192.168.0.1 fluentd[11111]: [error] Hello, world!' | nc -u -w0 127.0.0.1 5140
```

### Docker

A Docker file is provided for ease of installation wherever you may want to fun this.

1. Build the Docker image

    ```shell
    docker build -t s3-fluentd:latest s3-fluentd
    ```

2. Run it!

    In order for Docker to be able to write to S3, you'll need to either pass environment variables on to Docker or modify [s3-fluentd/fluent.conf](s3-fluentd/fluent.conf).

    ```shell
    docker run -it --rm \
      df--name s3-docker-fluent-logger \
      -p 5140:5140/udp \
      -v $(pwd)s3-fluentd/log:/fluentd/log \
      -e AWS_ACCESS_KEY_ID=<ACCESS_KEY> \
      -e AWS_SECRET_ACCESS_KEY=<SECRET_KEY>
      s3-fluentd:latest
    ```

By default, fluentd will listen for syslog connections on UDP port 5140.
Test it similar to the manual method.

### AWS Fargate

Coming soon... ?


### Query with Athena 

A table needs to be created in Athena that can query the Fluentd output data. This should be in the same region that your bucket is in for best performance and cost.

1. (Optional) Create a specific database for your logs

    ```sql
    CREATE DATABASE logs;
    ```

2. Create a table in Athena

    Make sure you provide the correct bucket and prefix in the `LOCATION` field.
    By default, we're going to create a table _without_ partitions, even though
    we're writing data with Hive-style partitions. This is for ease of use (no need to use `MSCK REPAIR TABLE` to discover partitions). But in a production environment, a partitioned table should be used.

    ```sql
    CREATE EXTERNAL TABLE syslog (
    host string,
    ident string,
    pid string,
    message string,
    time string
    )
    ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
    LOCATION 's3://<BUCKET>/syslog/';
    ```

This is an example of what the Fluentd data looks like:

```json
{
    "host":"192.168.0.1",
    "ident":"fluentd",
    "pid":"11111",
    "message":"[error] Hello, world!",
    "time":"2018-10-31T19:46:38-07:00"
}
```

## Research

### Testing

(for macOS)
```shell
echo '<13>'$(date -u +"%b %d %T")' 192.168.0.1 fluentd[11111]: [error] Hello, world!' | nc -u -w0 127.0.0.1 5140
```

### There may be different syslog patterns. 
Specifically for Ubiquiti: https://www.reddit.com/r/Ubiquiti/comments/3mkr85/syslog_message_format_for_uap/

Could use a grok parser: https://github.com/fluent/fluent-plugin-grok-parser
Along with multiple parser plugin: https://github.com/repeatedly/fluent-plugin-multi-format-parser

### Default fluentd output is kind of weird

```
2018-02-28T12:00:00-08:00	system.kern.info	{"host":"192.168.0.1","ident":"fluentd","pid":"11111","message":"[error] Hello, world!"}
```

Oh, that's because I have this: `<buffer tag,time>`

Nope. It's because I need to change the output format.

```
  <format>
    @type json
  </format>
```

### Flush mode in testing

Flush mode by default is lazy so things don't immediately get sent to s3
Can force it with the following in `buffer` section.
```
    flush_interval 30s
    flush_mode interval
```

Darn, can't use `timestamp` format:
```
HIVE_BAD_DATA: Error parsing field value '2018-10-31T21:16:29-07:00' for field 4: Timestamp format must be yyyy-mm-dd hh:mm:ss[.fffffffff]
```

## Fargate?

Maybe. ECS on EC2.
Def use instance creds: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html

Short-term `docker run -e AWS_ACCESS_KEY_ID=xyz -e AWS_SECRET_ACCESS_KEY=aaa myimage`

## Things you may want to tweak

### Buffer storage

Default, file. Optional, memory.

### Timekey

Defaults to hourly files.

### Path

Defaults to Hive-compatible year/month/day.