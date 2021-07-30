---
title: canal-adapter配置
date: 2021-02-22 16:51:07
categories:
  - 部署
  - 攻略
tags:
  - 服务器
---

参考：<https://blog.csdn.net/jcmj123456/article/details/109705562>

## MySQL

MySQL 需开启 binlog 写入功能，并配置 binlog-format 为 ROW 模式

```
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1 # 配置 MySQL replaction 需要
```
检查数据库是否打开上述开关  
```sql
show variables like '%log_bin%'
show variables like 'binlog_format%';
```

创建 canal 账号，用于 canal 连接 MySQL， 该账号必须具有作为 MySQL slave 的权限，如使用已有账户，可直接使用 grant 命令授权

```sql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

假设有数据库表 `product`

```

CREATE TABLE `product`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `title` varchar(255),
  `sub_title` varchar(255),
  PRIMARY KEY (`id`)
)
```



## Canal Server

使用 docker 启动，参考<https://github.com/alibaba/canal/wiki/Docker-QuickStart>

```shell
# 下载脚本
wget https://raw.githubusercontent.com/alibaba/canal/master/docker/run.sh 

# 构建一个destination name为test的队列
sh run.sh -e canal.auto.scan=false \
		  -e canal.destinations=test \
		  -e canal.instance.master.address=127.0.0.1:3306  \
		  -e canal.instance.dbUsername=canal  \
		  -e canal.instance.dbPassword=canal  \
		  -e canal.instance.connectionCharset=UTF-8 \
		  -e canal.instance.tsdb.enable=true \
		  -e canal.instance.gtidon=false
# 最终会变为：docker run -d -it -h 0 {上面所有的 -e 参数} --name=canal-server --net=host -m 4096m canal/canal-server
```



## Elastic 设置

先运行一个没有 xpack 功能的集群，版本是 7.10.2

创建数据库表 product 对应的索引，在 kibana 的 dev_tools 里运行：

```json
PUT product
{
  "mappings": {
    "properties": {
      "id": {
        "type": "integer"
      },
      "title": {
        "type": "text"
      },
      "sub_title": {
        "type": "text"
      }
    }
  }
}
```



## Canal-Adapter 部署

- 从 github 上下载并解压 release 包  <https://github.com/alibaba/canal/releases/download/canal-1.1.5/canal.adapter-1.1.5.tar.gz>

- 由于发布包里的 ` client-adapter.es7x`有 bug，会出现 com.alibaba.druid.pool.DruidDataSource cannot be cast to com.alibaba.druid.pool.DruidDataSource  的 bug，见 [issue 3144](https://github.com/alibaba/canal/issues/3144)，需要重新编译：

  - 下载源码 <https://github.com/alibaba/canal/archive/refs/tags/canal-1.1.5.tar.gz>
  - 修改 client-adapter/escore 目录下的 pom.xml，在 druid 的依赖定义增加`<scope>provided</scope>`
  - 在项目的**根目录**下编译。用新编译出来的 client-adapter.es7x-1.1.5-jar-with-dependencies.jar ，替换原来 plugin 目录下的

- 配置 canal-adapter 的 conf/application.yml

  ```yaml
  canal.conf:
    mode: tcp #tcp kafka rocketMQ rabbitMQ
    consumerProperties:
      # canal tcp consumer
      canal.tcp.server.host: 127.0.0.1:11111
    srcDataSources:
      defaultDS:
        url: jdbc:mysql://127.0.0.1:3306/{DBName}?useUnicode=true
        username: canal
        password: canal  
    canalAdapters:
    - instance: test # 对应启动 canal server 时的 canal.destinations
      groups:
      - groupId: g1
        outerAdapters:
        - name: logger
        
  # 重点在这里，输出到 es 的定义      
        - name: es7
          hosts: http://192.168.1.194:9200 #172.22.0.2:9300 # 127.0.0.1:9200 for rest mode
          properties:
            mode: rest #transport # or rest
            #security.auth: elastic:123456 #  only used for rest mode
            cluster.name: es-docker-cluster # 在启动 es 集群的 docker-compose.yml里有
  ```

- 上面在 outerAdapters 里定义了 es7 ，则会加载 conf/es7 文件夹里的 yml 文件，

  假设有 product.yml：

  ```yaml
  dataSourceKey: defaultDS
  destination: test 
  groupId: g1
  esMapping:
    _index: product
    _id: id
    sql: "select t.id, t.title, t.sub_title from product t"
    #  etlCondition: "where t.c_time>={}"
    commitBatch: 3000
  ```

- 第一次要手工全量导入

  ```shell
  curl http://127.0.0.1:8081/etl/es7/product.yml -X POST
  # 正常时会返回导入多少数据
  ```

  

- 修改数据库表内容，看 es 里的数据是否改变，在 kibana 的 dev_tools 里运行
  
  ```
  GET product/_search
  ```
  
  

