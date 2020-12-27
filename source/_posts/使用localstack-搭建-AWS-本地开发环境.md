---
title: 使用localstack 搭建 AWS 本地开发环境
date: 2020-12-27 14:45:59
tags: [AWS, localstack]
categories: [AWS]
---

![DBhQX3](https://cdn.jsdelivr.net/gh/Fatezhang/FigureCloud@master/uPic/DBhQX3.jpg)

## 前言

相信有很多同学公司项目都有用到 AWS 或者有同学在学习 AWS，我们知道像 AWS 这样的云服务，我们不太能很方便的去在本地开发的时候连接云上的服务，更何况 staging/production 环境还会有安全方面的考虑。那么当我们创建好一个项目的时候，我们如何去搭建其本地开发环境方便我们在本地开发调试呢？没错！使用 localstack！

__[LocalStack - A fully functional local AWS cloud stack](https://github.com/localstack/localstack)__

> ***LocalStack provides an easy-to-use test/mocking framework for developing Cloud applications.***
>
> ***Currently, the focus is primarily on supporting the AWS cloud stack.***

LocalStack 是提供给开发者一个方便去测试和 mock 服务的框架， 目前主要提供 AWS 云服务。

<!--more-->



## 如何使用？

### 例：使用 LocalStack 创建一个 SNS服务

想像我们有这样一个 SpringBoot 服务，提供接口在用户提交一条记录的时候给用户发邮件，一般情况这种业务我们都会将保存数据库和发邮件异步执行。使用 AWS 的话，我们就可以在保存数据库之后调用 SNS 服务，发布一个 `Event` 然后等待下游邮件服务订阅对应的 topic 然后消费。

那么如何创建呢？如下在 `docker-compose.yml` 文件中加入`localstack `，SERVICES 指定好 `sns`：

#### docker-compose.yml

```yml
version: "3.7"

services:
  localstack:
    image: localstack/localstack:0.12.1
    networks:
      - app_net
    ports:
      - "4566:4566"
    environment:
      - SERVICES=sns
      - DEFAULT_REGION=ap-southeast-2
      - DEBUG=1
    volumes:
      - ./auto/create-localstack-topic:/docker-entrypoint-initaws.d/create-localstack-topic.sh
```

#### create-localstack-topic.sh

此外，可以看到我们还 volume 了一个脚本进去，那就是创建 SNS 的脚本。实际上这个命令就是 aws cli，在其官网就可以找得到，我们只需要把 `aws` 换成 `awslocal`

```bash
#!/bin/bash

REGION=${DEFAULT_REGION:-ap-southeast-2}
TOPIC_NAME=demo-events-topic

awslocal sns create-topic --name=${TOPIC_NAME} --region "${REGION}"
```

#### 启动日志

当启动 `docker-compose up localstack` 的时候，就在该容器中创建好了 SNS 服务提供给我们使用。

```bash
sns_1            | Waiting for all LocalStack services to be ready
sns_1            | 2020-12-27 07:02:36,554 CRIT Supervisor is running as root.  Privileges were not dropped because no user is specified in the config file.  If you intend to run as root, you can set user=root in the config file to avoid this message.
sns_1            | 2020-12-27 07:02:36,559 INFO supervisord started with pid 15
sns_1            | 2020-12-27 07:02:37,566 INFO spawned: 'dashboard' with pid 21
sns_1            | 2020-12-27 07:02:37,571 INFO spawned: 'infra' with pid 22
sns_1            | 2020-12-27 07:02:37,577 INFO success: dashboard entered RUNNING state, process has stayed up for > than 0 seconds (startsecs)
sns_1            | 2020-12-27 07:02:37,577 INFO exited: dashboard (exit status 0; expected)
sns_1            | (. .venv/bin/activate; exec bin/localstack start --host)
sns_1            | 2020-12-27 07:02:38,591 INFO success: infra entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
sns_1            | LocalStack version: 0.12.1
sns_1            | Starting local dev environment. CTRL-C to quit.
sns_1            | 2020-12-27T07:02:39:DEBUG:bootstrap.py: Loading plugins - scope "services", module "localstack": <function register_localstack_plugins at 0x7f963f120f70>
sns_1            | Waiting for all LocalStack services to be ready
sns_1            | 2020-12-27T07:02:43:INFO:localstack.utils.analytics.profiler: Execution of "load_plugin_from_path" took 4333.9550495147705ms
sns_1            | 2020-12-27T07:02:43:INFO:localstack.utils.analytics.profiler: Execution of "load_plugins" took 4334.24186706543ms
sns_1            | Starting edge router (https port 4566)...
sns_1            | Starting mock SNS service on http port 4566 ...
sns_1            | 2020-12-27T07:02:45:INFO:localstack.utils.analytics.profiler: Execution of "prepare_environment" took 2061.4540576934814ms
sns_1            | 2020-12-27T07:02:45:INFO:localstack.multiserver: Starting multi API server process on port 59903
sns_1            | [2020-12-27 07:02:45 +0000] [23] [INFO] Running on https://0.0.0.0:4566 (CTRL + C to quit)
sns_1            | 2020-12-27T07:02:45:INFO:hypercorn.error: Running on https://0.0.0.0:4566 (CTRL + C to quit)
sns_1            | [2020-12-27 07:02:45 +0000] [23] [INFO] Running on http://0.0.0.0:59903 (CTRL + C to quit)
sns_1            | 2020-12-27T07:02:45:INFO:hypercorn.error: Running on http://0.0.0.0:59903 (CTRL + C to quit)
sns_1            | 2020-12-27 07:02:45,824:API:  * Running on http://0.0.0.0:57589/ (Press CTRL+C to quit)
sns_1            | Waiting for all LocalStack services to be ready
sns_1            | Ready.
sns_1            | 2020-12-27T07:02:50:INFO:localstack.utils.analytics.profiler: Execution of "start_api_services" took 5102.221965789795ms
sns_1            | /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initaws.d/create-localstack-topic.sh
sns_1            | {
sns_1            |     "TopicArn": "arn:aws:sns:ap-southeast-2:000000000000:demo-events-topic"
sns_1            | }
```



#### 在命令行调用

```bash
aws --endpoint-url=http://localhost:4566 sns publish --topic-arn arn:aws:sns:ap-southeast-2:000000000000:demo-events-topic --region ap-southeast-2 --message "Hello SNS"
```

注意，在本地使用 aws 命令调用 localstack 中的服务的时候，需要覆盖`endpoint-url`, 否则回去拿着 credentials 调用实际环境的服务。

#### 在 SpringBoot 中使用的注意点

在 SpringBoot 或者其他代码库（如node）中使用的话，可以根据不同的环境创建不同的 `SNSClient`， 本地环境的注意要覆盖 endpoint：

```java
@Configuration
@Profile({"local", "docker"})
public class LocalSnsClientConfiguration {

    @Value("${aws.sns.endpoint}")
    private String awsSnsEndpoint;

    @Bean
    public SnsClient snsClient() {
        var clientBuilder = SnsClient.builder();
        if (!Strings.isNullOrEmpty(awsSnsEndpoint)) {
            clientBuilder.endpointOverride(URI.create(awsSnsEndpoint));
        }
        return clientBuilder.build();
    }
}
```

### 启动多个服务

上面只是一个启动 SNS 服务的例子，实际使用中，我们都会多种服务结合使用。比如会有一个 SQS 服务， 订阅了 SNS 的 topic，然后去 trigger 一个 lambda，执行相应的一些任务，那么如何在本地实现这些服务的相互订阅与触发呢？实际上只需要在一个 localstack 中启动多个服务然后执行一些脚本建立之间的关系（具体命令和 aws cli 一样）就可以了，如下：

#### docker-compose.yml

```yaml
version: "3.7"

services:
  localstack:
    image: localstack/localstack:0.12.1
    privileged: true
    container_name: localstack
    networks:
      - app_net
    ports:
      - "4566:4566"
    environment:
      - SERVICES=sns,sqs,kms,cloudwatch,lambda
      - DEFAULT_REGION=ap-southeast-2
      - LAMBDA_EXECUTOR=docker-reuse
      - LAMBDA_REMOTE_DOCKER=false
      - LAMBDA_DOCKER_NETWORK=host
      - DEBUG=1
      - HOST_TMP_FOLDER=${TMPDIR}
      - DOCKER_HOST=unix:///var/run/docker.sock
      - LOCAL_CODE_PATH=${PWD}
    volumes:
      - ${TMPDIR:-/tmp/localstack}:/tmp/localstack
      - /var/run/docker.sock:/var/run/docker.sock
      - ./auto/create-localstack:/docker-entrypoint-initaws.d/create-localstack.sh
      - ./kms/kms_seed.yaml:/init/seed.yaml

networks:
  app_net:

```

#### kms_seed.yaml 

我在这里启动了 sns,sqs,kms,cloudwatch,lambda。 比较值得一说的除了在本地访问 sqs, sns, kms 等需要覆盖掉 `endpoint-url`之外， 在本地使用 kms 还需要指定一个 `seed.yml` 来用它进行加解密。

```yaml
Keys:
  Symmetric:
    Aes:
      - Metadata:
          KeyId: 832ac356-3c82-4c4d-a3dc-7489da152197
        BackingKeys:
          - 2bdaead27fe7da2de47945d34cd6d79e36494e73802f3cd3869f1d2cb0b5d74c
Aliases:
  - AliasName: alias/testing
    TargetKeyId: 832ac356-3c82-4c4d-a3dc-7489da152197

```

#### 创建脚本 `create-localstack.sh` 

```shell
#!/bin/bash

QUEUE_NAME=demo-queue
TOPIC_NAME=demo-topic
FUNCTION_NAME=demo-function
APP_ENV=dev

awslocal sns create-topic --name=${TOPIC_NAME}
awslocal sqs create-queue --queue-name=${QUEUE_NAME}
awslocal sns subscribe \
    --topic-arn arn:aws:sns:ap-southeast-2:000000000000:${TOPIC_NAME} \
    --protocol sqs \
    --notification-endpoint http://localhost:4566/000000000000/${QUEUE_NAME}

awslocal lambda create-function \
    --code S3Bucket="__local__",S3Key="${LOCAL_CODE_PATH}" \
    --function-name ${FUNCTION_NAME} \
    --runtime nodejs12.x \
    --timeout 5 \
    --handler dist/index.handler \
    --role dev \
    --environment "{\"Variables\":{\"APP_ENV\":\"${APP_ENV}\"}}"

awslocal lambda create-event-source-mapping \
    --event-source-arn arn:aws:sqs:ap-southeast-2:000000000000:${QUEUE_NAME} \
    --function-name ${FUNCTION_NAME} \
    --enabled

```

#### 本地启动

直接运行 `docker-compose up localstack`

#### 本地执行

现在在命令行发送一条 SNS 的消息，就可以 trigger 我们的 Lambda 执行了。

```shell
aws --endpoint-url=http://localhost:4566 sns publish --topic-arn arn:aws:sns:ap-southeast-2:000000000000:demo-topic --region ap-southeast-2 --message "Hello SNS - SQS - Lambda"
```

在 Lambda 的 index.ts 写一个 `handler()` 方法：

```typescript
require('./overwriteAwsLocalEndpoint'); //overwrite aws local endpoint,Please keep it here.
import { SQSEvent, SQSHandler } from 'aws-lambda';

export const handler: SQSHandler = (event: SQSEvent) => {
  console.log(JSON.stringify(event.Records));
}
```

打印一下 SQS 的消息体。

__结果如下__

```bash
localstack    | > START RequestId: ce5ae5ff-054d-16e0-dc62-71161118d3bd Version: $LATEST
localstack    | > 2020-12-27T09:32:56.373Z	ce5ae5ff-054d-16e0-dc62-71161118d3bd	INFO	[{"body":"{\"Type\": \"Notification\", \"MessageId\": \"04c12e03-66d0-474a-a60a-f0b3c2451456\", \"Token\": null, \"TopicArn\": \"arn:aws:sns:ap-southeast-2:000000000000:demo-topic\", \"Message\": \"Hello SNS - SQS - Lambda\", \"SubscribeURL\": null, \"Timestamp\": \"2020-12-27T09:32:52.202Z\", \"SignatureVersion\": \"1\", \"Signature\": \"EXAMPLEpH+..\", \"SigningCertURL\": \"https://sns.us-east-1.amazonaws.com/SimpleNotificationService-0000000000000000000000.pem\"}","receiptHandle":"exexifyylldwuznxlicibcanaqvcplpaeoztcdlltkzbsuvwiifvlyixrxwuzrmumlmkggofmiencdxilzoaluyreszdppsbycpxcowwvmeiieeplulkitfztfxzkjazucucauhuobpvlzdcnjdcmygqvbrouxkxoggcfryzqtibyquhikawczuif","md5OfBody":"d96df71c445e9282ed4c2fefbf4c8ca1","eventSourceARN":"arn:aws:sqs:ap-southeast-2:000000000000:demo-queue","eventSource":"aws:sqs","awsRegion":"ap-southeast-2","messageId":"a59f7c57-651b-54f6-70bd-a2933fa57099","attributes":{"SenderId":"AIDAIT2UOQQY3AUEKVGXU","SentTimestamp":"1609061572241","ApproximateReceiveCount":"1","ApproximateFirstReceiveTimestamp":"1609061572312"},"messageAttributes":{},"md5OfMessageAttributes":null,"sqs":true}]
localstack    | > END RequestId: ce5ae5ff-054d-16e0-dc62-71161118d3bd
localstack    | > REPORT RequestId: ce5ae5ff-054d-16e0-dc62-71161118d3bd	Init Duration: 3381.65 ms	Duration: 13.13 ms	Billed Duration: 100 ms	Memory Size: 1536 MB	Max Memory Used: 55 MB
```

### 关于 LocalStack 中 Lambda 的使用

在本地创建 Lambda 运行环境是我觉得诸多 service 中比较麻烦的一个，以下是官方对于 Lambda 的创建时候的[配置详解](https://github.com/localstack/localstack#configurations)

```markdown
STEPFUNCTIONS_LAMBDA_ENDPOINT: URL to use as the Lambda service endpoint in Step Functions. By default this is the LocalStack Lambda endpoint. Use default to select the original AWS Lambda endpoint.

LAMBDA_EXECUTOR: Method to use for executing Lambda functions. Possible values are:

	- local: run Lambda functions in a temporary directory on the local machine
	- docker: run each function invocation in a separate Docker container
	- docker-reuse: create one Docker container per function and reuse it across invocations
	For docker and docker-reuse, if LocalStack itself is started inside Docker, then the docker command needs to be available inside the container (usually requires to run the container in privileged mode). Default is docker, fallback to local if Docker is not available.

LAMBDA_REMOTE_DOCKER: determines whether Lambda code is copied or mounted into containers. Possible values are:

	- true (default): your Lambda function definitions will be passed to the container by copying the zip file (potentially slower). It allows for remote execution, where the host and the client are not on the same machine.
	- false: your Lambda function definitions will be passed to the container by mounting a volume (potentially faster). This requires to have the Docker client and the Docker host on the same machine.

LAMBDA_DOCKER_NETWORK: Optional Docker network for the container running your lambda function.

LAMBDA_DOCKER_DNS: Optional DNS server for the container running your lambda function.

LAMBDA_CONTAINER_REGISTRY: Use an alternative docker registry to pull lambda execution containers (default: lambci/lambda).

LAMBDA_REMOVE_CONTAINERS: Whether to remove containers after Lambdas finished executing (default: true).
```



## FAQ

1. 为什么我使用 SNS、KMS 总是报一些 client 的 credentials 的错误？
   - 因为没有覆盖 本地环境需要的 endpoint-url， 参考本文中的解释
2. 为什么给 SNS 发消息成功了却没有触发到 Lambda？
   - 请检查你的创建脚本，确保你的 SQS 订阅了 SNS 的对应 topic，SQS 有能够触发 Lambda 的 Mapping
3. 本例代码库地址？
   - [Fatezhang/aws-localstack-demo](https://github.com/Fatezhang/aws-localstack-demo)

## 最后

如果你在使用 localstack 的时候遇到了什么问题，欢迎告诉我一起研究讨论。