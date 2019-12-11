---
title: 如何在 SpringCloud 微服务项目中一键部署 docker 启动
date: 2019-12-11 17:26:46
categories: [SpringCloud]
tags: [spring,springboot,SpringCloud,docker,微服务]
---

![](docker.jpg)

<!--more-->

## 如何用 docker 一键启动 SpringCloud 微服务？

### 为什么要使用 docker 启动我们的微服务？

原始传统的部署方式为我们带来了太多问题，比如当你的服务在测试环境正运行的正常，但是部署到生产环境就会出现一大堆未知问题，你查了好久，debug 代码之后发现，仅仅是一个环境变量没有设置正确，导致产生了目前这么一大串问题，浪费了你的时间，拖延了项目上线的时间，甚至产生了严重的生产事故。这个时候，我们就需要一个机制来保证我们的代码在部署到本地、测试以及生产的时候，我们的代码不会受到环境不一致的影响而导致我们部署失败！

还有就是我们可以使用 docker 打包镜像然后使用 docker-compose 编排容器一键启动，让运维自动化，极大的节省我们的工作时间。

### 在 Spring-Cloud 微服务项目中如何使用 DOCKER？

#### 首先我们需要将我们的每一个服务打包成镜像

例如我的[ Scaffold-Cloud 项目中的 eureka 服务的 pom 文件](https://github.com/Fatezhang/scaffold-cloud/blob/a6dd55772f59eecd99b099e013ce2ef4470cec91/scaffold-eureka/pom.xml#L43)(点击链接跳转到 github 以下代码行)如下：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <dependencies>
                <dependency>
                    <groupId>javax.activation</groupId>
                    <artifactId>activation</artifactId>
                    <version>1.1.1</version>
                </dependency>
            </dependencies>
            <version>0.4.13</version>
            <configuration>
                <imageName>scaffold-cloud/${project.artifactId}:${project.version}</imageName>
                <forceTags>true</forceTags>
                <baseImage>java</baseImage>
                <resources>
                    <resource>
                        <targetPath>/</targetPath>
                        <directory>${project.build.directory}</directory>
                        <include>${project.build.finalName}.jar</include>
                    </resource>
                </resources>
            </configuration>
        </plugin>
    </plugins>
</build>
```

然后其他模块也一样，在 pom 文件中加入以上 maven 插件，完整项目代码如下(觉得好用 star✨一下哦)：

[https://github.com/Fatezhang/scaffold-cloud/](https://github.com/Fatezhang/scaffold-cloud)

#### 使用脚本一键创建 docker 镜像

```shell
#!/bin/bash

echo "============start to package with maven and recreate docker image=============="

SERVICE_FOLDERS=(
  scaffold-eureka
  scaffold-zuul
  scaffold-tx-manager
  scaffold-business/scaffold-business-sys-service
  scaffold-business/scaffold-business-job-service
  scaffold-business/scaffold-business-thirdparty-service
  scaffold-route/scaffold-route-app
  scaffold-route/scaffold-route-openapi
  scaffold-route/scaffold-route-operate
)
path=
for (( i = 0; i < ${#SERVICE_FOLDERS[@]}; i++ )); do
    path=${SERVICE_FOLDERS[${i}]}
    echo "进入目录 >>>> cd ${path}"
    cd "${path}" || exit
    pwd
    mvn clean package docker:build -Pdocker
    cd - || exit
done
echo "============                      create end                     =============="
```

#### 编排 docker-compose 启动所有微服务

然后只需要将所有镜像编排在 docker-compose.yml 文件中即可，其中所有的容器都在同一个网络中以访问到对方服务，如下：

```yaml
networks:
  - scaffold-cloud
```
完整的 docker-compose.yml 文件如下（注意有一个`wait-for-it.sh`）：
```yaml
version: "3.7"

services:
  micro-sys:
    entrypoint: ./wait-for-it.sh tx_manager:7970 'java -jar scaffold-business-sys-service-1.0-SNAPSHOT.jar'
    image: scaffold-cloud/scaffold-business-sys-service:1.0-SNAPSHOT
    volumes:
      - /tmp:/tmp
      - ./.scripts/wait-for-it.sh:/wait-for-it.sh
    ports:
      - 8750:8750
    networks:
      - scaffold-cloud
    depends_on:
      - redis
      - eureka
      - mysql
      - rmqbroker
      - rmqnamesrv
      - tx_manager
  route-operate:
    entrypoint: ./wait-for-it.sh micro-sys:8750 'java -jar scaffold-route-operate-1.0-SNAPSHOT.jar'
    image: scaffold-cloud/scaffold-route-operate:1.0-SNAPSHOT
    volumes:
      - /tmp:/tmp
      - ./.scripts/wait-for-it.sh:/wait-for-it.sh
    ports:
      - 8850:8850
    networks:
      - scaffold-cloud
    depends_on:
      - micro-sys
  eureka:
    entrypoint: java -jar scaffold-eureka-1.0-SNAPSHOT.jar
    image: scaffold-cloud/scaffold-eureka:1.0-SNAPSHOT
    volumes:
      - /tmp:/tmp
    ports:
      - 8761:8761
    networks:
      - scaffold-cloud
  zuul:
    entrypoint: ./wait-for-it.sh eureka:8761 'java -jar scaffold-zuul-1.0-SNAPSHOT.jar'
    image: scaffold-cloud/scaffold-zuul:1.0-SNAPSHOT
    volumes:
      - /tmp:/tmp
      - ./.scripts/wait-for-it.sh:/wait-for-it.sh
    ports:
      - 8861:8861
    networks:
      - scaffold-cloud
    depends_on:
      - eureka
  tx_manager:
    entrypoint: ./wait-for-it.sh mysql:3306,redis:6379 'java -jar scaffold-txmanager-1.0-SNAPSHOT.jar'
    image: scaffold-cloud/scaffold-txmanager:1.0-SNAPSHOT
    volumes:
      - /tmp:/tmp
      - ./.scripts/wait-for-it.sh:/wait-for-it.sh
    ports:
      - 8890:8890
      - 7970:7970
    networks:
      - scaffold-cloud
    depends_on:
      - mysql
      - redis
      - eureka
  mysql:
    image: mysql/mysql-server:5.7
    ports:
      - 3306:3306
    volumes:
      - ./mysql/data:/var/lib/mysql
      - ./mysql/config/my.cnf:/etc/my.cnf
      - ./mysql/init:/docker-entrypoint-initdb.d/
    restart: always
    networks:
      - scaffold-cloud
  redis:
    image: redis:latest
    restart: always
    ports:
      - 6379:6379
    command: redis-server --requirepass pwd8ok8
    networks:
      - scaffold-cloud
  rmqnamesrv:
    image: foxiswho/rocketmq:server-4.5.2
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    volumes:
      - ./rmq/logs:/opt/logs
      - ./rmq/store:/opt/store
    environment:
      JAVA_OPT_EXT: "-Duser.home=/opt -Xms128m -Xmx128m -Xmn128m"
    networks:
      - scaffold-cloud
  rmqbroker:
    image: foxiswho/rocketmq:broker-4.5.2
    container_name: rmqbroker
    ports:
      - 10909:10909
      - 10911:10911
    volumes:
      - ./rmq/logs:/opt/logs
      - ./rmq/store:/opt/store
      - ./rmq/brokerconf/broker.conf:/etc/rocketmq/broker.conf
    environment:
      JAVA_OPT_EXT: "-Duser.home=/opt -server -Xms128m -Xmx128m -Xmn128m"
    command: ["/bin/bash","mqbroker","-c","/etc/rocketmq/broker.conf","-n","rmqnamesrv:9876","autoCreateTopicEnable=true"]
    depends_on:
      - rmqnamesrv
    networks:
      - scaffold-cloud

  rmqconsole:
    image: styletang/rocketmq-console-ng
    container_name: rmqconsole
    ports:
      - 8180:8080
    environment:
      JAVA_OPTS: "-Drocketmq.namesrv.addr=rmqnamesrv:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false"
    depends_on:
      - rmqnamesrv
    networks:
      - scaffold-cloud
networks:
  scaffold-cloud:
    name: scaffold-cloud
    driver: bridge
```

#### 如何在启动所有微服务的时候按顺序启动呢？

我们知道，SpringCloud 微服务中各个服务之间可能存在依赖关系，比如 tx-manager 服务需要等待 MySQL 和 Redis 都启动完成之后才能够顺利启动，但是 docker-compose.yml 文件中的 `depends_on`只能等待 docker 容器启动成功，而不能等待 docker 容器中的服务启动完成！

docker 官方给出的方案是使用wait-for-it.sh脚本，在容器启动之前执行脚本，判断依赖的服务和端口是否启动成功，然后再执行自己服务的启动命令。

所以我们也是这样去写一个脚本：`wait-for-it.sh`如下

```shell
#!/bin/bash

# shellcheck disable=SC2223
: ${SLEEP_SECOND:=2}

wait_for() {
  echo Waiting for $1 on port $2... ...
  # shellcheck disable=SC2086
  # shellcheck disable=SC2188
  while ! </dev/tcp/$1/$2; do
    echo Waiting for $1 on port $2... ...
    sleep $SLEEP_SECOND
  done
  echo Server:"$1" on port:"$2" is running... ...
}

SER_STRS=$1

# shellcheck disable=SC2207
# shellcheck disable=SC2006
SERVICES_PORTS=($(echo "$SER_STRS" | tr ',' ' '))

THEN_COMMAND=$2

for ((i = 0; i < ${#SERVICES_PORTS[@]}; i++)); do
  SERVICE_PORT=${SERVICES_PORTS[${i}]}
  # shellcheck disable=SC2207
  # shellcheck disable=SC2006
  array=($(echo "$SERVICE_PORT" | tr ':' ' '))
  servername=${array[0]}
  serverport=${array[1]}
  wait_for "$servername" "${serverport}"
done
echo "start to run command: $THEN_COMMAND"
# shellcheck disable=SC2004
if [ "$THEN_COMMAND" ]; then
  eval "$THEN_COMMAND"
else
  # shellcheck disable=SC2102
  echo command: [$THEN_COMMAND] is invalid... ...
fi
```

上述脚本的作用就是等待启动的服务以及端口启动成功，没有成功的话就继续 wait 等待，默认等待 2 秒，启动成功的话就执行自己的启动命令。

使用方式如下：

```yaml
tx_manager:
    entrypoint: ./wait-for-it.sh mysql:3306,redis:6379 'java -jar scaffold-txmanager-1.0-SNAPSHOT.jar'
```

所以，以上就是使用 docker 启动 SpringCloud 微服务的全部内容，如果你觉得有用，希望能够给我的项目https://github.com/Fatezhang/scaffold-cloud/点个star 哦~

而且你也可以给我提 issue，提 PR，欢迎一起交流学习，一起开发~