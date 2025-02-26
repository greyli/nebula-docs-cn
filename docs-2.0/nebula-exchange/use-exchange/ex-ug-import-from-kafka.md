# 导入Kafka数据

本文简单说明如何使用Exchange将存储在Kafka上的数据导入Nebula Graph。

## 环境配置

本文示例在MacOS下完成，以下是相关的环境配置信息：

- 硬件规格：
  - CPU：1.7 GHz Quad-Core Intel Core i7
  - 内存：16 GB

- Spark：2.4.7，单机版

- Nebula Graph：{{nebula.release}}。使用[Docker Compose部署](../../4.deployment-and-installation/2.compile-and-install-nebula-graph/3.deploy-nebula-graph-with-docker-compose.md)。

## 前提条件

开始导入数据之前，用户需要确认以下信息：

- 已经[安装部署Nebula Graph](../../4.deployment-and-installation/2.compile-and-install-nebula-graph/2.install-nebula-graph-by-rpm-or-deb.md)并获取如下信息：

  - Graph服务和Meta服务的的IP地址和端口。

  - 拥有Nebula Graph写权限的用户名和密码。

- 已经编译Exchange。详情请参见[编译Exchange](../ex-ug-compile.md)。本示例中使用Exchange {{exchange.release}}。

- 已经安装Spark。

- 了解Nebula Graph中创建Schema的信息，包括Tag和Edge type的名称、属性等。

- 已经安装并开启Kafka服务。

## 操作步骤

### 步骤 1：在Nebula Graph中创建Schema

分析数据，按以下步骤在Nebula Graph中创建Schema：

1. 确认Schema要素。Nebula Graph中的Schema要素如下表所示。

    | 要素  | 名称 | 属性 |
    | :--- | :--- | :--- |
    | Tag | `player` | `name string, age int` |
    | Tag | `team` | `name string` |
    | Edge Type | `follow` | `degree int` |
    | Edge Type | `serve` | `start_year int, end_year int` |

2. 在Nebula Graph中创建一个图空间**basketballplayer**，并创建一个Schema，如下所示。

    ```ngql
    ## 创建图空间
    nebula> CREATE SPACE basketballplayer \
            (partition_num = 10, \
            replica_factor = 1, \
            vid_type = FIXED_STRING(30));
    
    ## 选择图空间basketballplayer
    nebula> USE basketballplayer;
    
    ## 创建Tag player
    nebula> CREATE TAG player(name string, age int);
    
    ## 创建Tag team
    nebula> CREATE TAG team(name string);
    
    ## 创建Edge type follow
    nebula> CREATE EDGE follow(degree int);

    ## 创建Edge type serve
    nebula> CREATE EDGE serve(start_year int, end_year int);
    ```

更多信息，请参见[快速开始](../../2.quick-start/1.quick-start-workflow.md)。

### 步骤 2：修改配置文件

!!! note

    如果部分数据存储在Kafka的value域内，需要自行修改源码，从Kafka中获取value域，将value通过from_json函数解析，然后作为Dataframe返回。

编译Exchange后，复制`target/classes/application.conf`文件设置Kafka数据源相关的配置。在本示例中，复制的文件名为`kafka_application.conf`。各个配置项的详细说明请参见[配置说明](../parameter-reference/ex-ug-parameter.md)。

```conf
{
  # Spark相关配置
  spark: {
    app: {
      name: Nebula Exchange {{exchange.release}}
    }
    driver: {
      cores: 1
      maxResultSize: 1G
    }
    cores {
      max: 16
    }
  }


  # Nebula Graph相关配置
  nebula: {
    address:{
      # 以下为Nebula Graph的Graph服务和Meta服务所在机器的IP地址及端口。
      # 如果有多个地址，格式为 "ip1:port","ip2:port","ip3:port"。
      # 不同地址之间以英文逗号 (,) 隔开。
      graph:["127.0.0.1:9669"]
      meta:["127.0.0.1:9559"]
    }
    # 填写的账号必须拥有Nebula Graph相应图空间的写数据权限。
    user: root
    pswd: nebula
    # 填写Nebula Graph中需要写入数据的图空间名称。
    space: basketballplayer
    connection {
      timeout: 3000
      retry: 3
    }
    execution {
      retry: 3
    }
    error: {
      max: 32
      output: /tmp/errors
    }
    rate: {
      limit: 1024
      timeout: 1000
    }
  }
  # 处理点
  tags: [
    # 设置Tag player相关信息。
    {
      # Nebula Graph中对应的Tag名称。
      name: player
      type: {
        # 指定数据源文件格式，设置为Kafka。
        source: kafka
        # 指定如何将点数据导入Nebula Graph：Client或SST。
        sink: client
      }
      # Kafka服务器地址。
      service: "127.0.0.1:9092"
      # 消息类别。
      topic: "topic_name1"

      # Kafka数据有固定的域名称：key、value、topic、partition、offset、timestamp、timestampType。
      # Spark读取为DataFrame后，如果需要指定多个字段，用英文逗号（,）隔开。
      # 在fields里指定字段名称，例如用key对应Nebula中的name， value对应Nebula中的age，示例如下：
      fields: [key,value]
      nebula.fields: [name,age]

      # 指定表中某一列数据为Nebula Graph中点VID的来源。
      # 这里的值key和上面的key重复，表示key既作为VID，也作为属性name。
      vertex:{
          field:key
      }


      # 单批次写入 Nebula Graph 的数据条数。
      batch: 10

      # Spark 分区数量
      partition: 10
      # 读取消息的间隔。单位：秒。
      interval.seconds: 10
    }
    # 设置Tag team相关信息。
    {
      name: team
      type: {
        source: kafka
        sink: client
      }
      service: "127.0.0.1:9092"
      topic: "topic_name2"
      fields: [key]
      nebula.fields: [name]
      vertex:{
          field:key
      }
      batch: 10
      partition: 10
      interval.seconds: 10
    }

  ]

  # 处理边数据
  edges: [
    # 设置Edge type follow相关信息
    {
      # Nebula Graph中对应的Edge type名称。
      name: follow

      type: {
        # 指定数据源文件格式，设置为Kafka。
        source: kafka

        # 指定边数据导入Nebula Graph的方式，
        # 指定如何将点数据导入Nebula Graph：Client或SST。
        sink: client
      }

      # Kafka服务器地址。
      service: "127.0.0.1:9092"
      # 消息类别。
      topic: "topic_name3"

      # Kafka数据有固定的域名称：key、value、topic、partition、offset、timestamp、timestampType。
      # Spark读取为DataFrame后，如果需要指定多个字段，用英文逗号（,）隔开。
      # 在fields里指定字段名称，例如用key对应Nebula中的degree，示例如下：
      fields: [key]
      nebula.fields: [degree]

      # 在source里，将topic中某一列作为边的起始点数据源。
      # 在target里，将topic中某一列作为边的目的点数据源。
      source:{
          field:timestamp
      }


      target:{
          field:offset
      }


      # 单批次写入 Nebula Graph 的数据条数。
      batch: 10

      # Spark 分区数量
      partition: 10

      # 读取消息的间隔。单位：秒。
      interval.seconds: 10
    }

    # 设置Edge type serve相关信息
    {
      name: serve
      type: {
        source: kafka
        sink: client
      }
      service: "127.0.0.1:9092"
      topic: "topic_name4"

      fields: [timestamp,offset]
      nebula.fields: [start_year,end_year]
      source:{
          field:key
      }

      target:{
          field:value
      }

      batch: 10
      partition: 10
      interval.seconds: 10
    }
  ]
}
```

### 步骤 3：向Nebula Graph导入数据

运行如下命令将Kafka数据导入到Nebula Graph中。关于参数的说明，请参见[导入命令参数](../parameter-reference/ex-ug-para-import-command.md)。

```bash
${SPARK_HOME}/bin/spark-submit --master "local" --class com.vesoft.nebula.exchange.Exchange <nebula-exchange-{{exchange.release}}.jar_path> -c <kafka_application.conf_path>
```

!!! note

    JAR包有两种获取方式：[自行编译](../ex-ug-compile.md)或者从maven仓库下载。

示例：

```bash
${SPARK_HOME}/bin/spark-submit  --master "local" --class com.vesoft.nebula.exchange.Exchange  /root/nebula-spark-utils/nebula-exchange/target/nebula-exchange-{{exchange.release}}.jar  -c /root/nebula-spark-utils/nebula-exchange/target/classes/kafka_application.conf
```

用户可以在返回信息中搜索`batchSuccess.<tag_name/edge_name>`，确认成功的数量。例如`batchSuccess.follow: 300`。

### 步骤 4：（可选）验证数据

用户可以在Nebula Graph客户端（例如Nebula Graph Studio）中执行查询语句，确认数据是否已导入。例如：

```ngql
GO FROM "player100" OVER follow;
```

用户也可以使用命令[`SHOW STATS`](../../3.ngql-guide/7.general-query-statements/6.show/14.show-stats.md)查看统计数据。

### 步骤 5：（如有）在Nebula Graph中重建索引

导入数据后，用户可以在Nebula Graph中重新创建并重建索引。详情请参见[索引介绍](../../3.ngql-guide/14.native-index-statements/README.md)。
