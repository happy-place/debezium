# 数据同步工具 debezium

## 简介
```aidl
Debezium是用于捕获变更数据的开源分布式平台。可以响应数据库的所有插入，更新和删除操作。Debezium依赖于kafka上，所以在安装Debezium时需要提前
安装好Zookeeper，Kafka，以及Kakfa Connect。

与canal、maxwell区别
1.canal、maxwell需要在my.cnf配置文件注册需要同步的库，devezium无需声明，在注册同步链路是，下发对谁同步，就抓取谁的binlog,只需要权限到位即可工作；
2.canal 元数据是自己管理，maxwell元数据交给外置mysql维护，debezium元数据交给kafka维护；
3.canal 中一条sql操作多行，则以集合形式收集binlog封装到json，且所有字段全部都是字符串形式，因此必须带有类型信息。maxwell一条sql操作了n行，
对应则有n行binlog json，数据类型为原始数据类型，没有类型信息，因此比较轻。debezium 受影响每一行，对应一条binlog-json，且带有数据类型，还有其他额外schema信息，
数据类型为原始类型，总体而言比较重。
4.canal、maxwell 都是以库为单位同步binlog，因此binlog是混合上报的，debezium是以表为单位同步数据，一个表对应一个topic；
5.flink-cdc 添加集成debezium，但在实际使用时，往往按maxwell风格重写schema解析器。
```

## Kafka Connect
```aidl
Kafka Connect用于在Apache Kafka和其他系统之间可扩展且可靠地数据流传输数据的工具，连接器可以轻松地将大量数据导入或导出。
Kafka Connect当前支持两种模式，standalone和distributed两种模式。standalone主要用于入门测试，所以我们来实现distributed模式。
官网地址：https://kafka.apache.org/documentation.html#connect
      
Distributed，分布式模式可以在处理工作中自动平衡，允许动态扩展或缩减，并在活动任务以及配置和偏移量提交数据中提供容错能力。和standalone模式非常类似，
最大区别在于启动的类和配置参数，参数决定了Kafka Connect流程如果存储偏移量，如何分配工作，在分布式模式下，Kafka Connect将偏移量，配置和任务状态存
储在topic中。建议手动创建topic指定分区数，也可以通过配置文件参数自动创建topic。

参数配置：
group.id 默认connect-cluster 集群的唯一名称，不能重名，用于形成connect集群组
config.storage.topic 用于存储kafka connector和任务配置信息的topic。
offset.storage.topic 用于存储偏移量的topic。
status.storage.topic 用于存储状态的topic。
```

## 配置
```aidl
cd /opt/softwares/kafka_2.11-0.11.0.0
vim connect-distributed.properties
-----------------------------------------------------------
bootstrap.servers=hadoop01:9092,hadoop02:9092,hadoop03:9092
-----------------------------------------------------------
```

## 安装 debezium 插件
```aidl
cd /opt/softwares/kafka_2.11-0.11.0.0
mkdir plugins

tar -zxvf debezium-connector-mysql-1.2.0.Final-plugin.tar.gz -C /opt/softwares/kafka_2.11-0.11.0.0/plugins

vi config/connect-distributed-mysql.properties 
-----------------------------------------------------------
plugin.path=/opt/softwares/kafka_2.11-0.11.0.0/plugins
-----------------------------------------------------------
```

## 分发
```aidl
cd /opt/softwares/kafka_2.11-0.11.0.0
xsync plugins
xsync config/connect-distributed-mysql.properties 
```

## 辅助脚本
* 启动Connector脚本
```aidl
vi  bin/kafkaConnectorStart
-----------------------------------------------------------
#!/bin/bash

curr_dir=`pwd`

KAFKA_HOME=/opt/softwares/kafka_2.11-0.11.0.0
ssh admin@hadoop01 "$KAFKA_HOME/bin/connect-distributed.sh -daemon $KAFKA_HOME/config/connect-distributed.properties"
ssh admin@hadoop02 "$KAFKA_HOME/bin/connect-distributed.sh -daemon $KAFKA_HOME/config/connect-distributed.properties"
ssh admin@hadoop03 "$KAFKA_HOME/bin/connect-distributed.sh -daemon $KAFKA_HOME/config/connect-distributed.properties"

xcall jps

cd $curr_dir
-----------------------------------------------------------
```

* 停止Connector脚本
```aidl
vi  bin/kafkaConnectorStop
-----------------------------------------------------------
#!/bin/bash

curr_dir=`pwd`

KAFKA_HOME=/opt/softwares/kafka_2.11-0.11.0.0

ssh admin@hadoop01 "ps -ef | grep ConnectDistributed | grep -v grep | awk -F ' ' '{print $2}' | xargs kill -s 9"
ssh admin@hadoop02 "ps -ef | grep ConnectDistributed | grep -v grep | awk -F ' ' '{print $2}' | xargs kill -s 9"
ssh admin@hadoop03 "ps -ef | grep ConnectDistributed | grep -v grep | awk -F ' ' '{print $2}' | xargs kill -s 9"

xcall jps

cd $curr_dir
-----------------------------------------------------------
```

* kafka 管理脚本
```aidl
vi  bin/kafka
-----------------------------------------------------------
#!/bin/bash

old=`pwd`
new=$KAFKA_HOME
cd $new
zks='hadoop01:2181,hadoop02:2181,hadoop03:2181'
brks='hadoop01:9092,hadoop02:9092,hadoop03:9092'

case $1 in
    list)
        echo "kafka-topics.sh --zookeeper $zks --list"
		kafka-topics.sh --zookeeper $zks --list
        ;;
    create)
        echo "kafka-topics.sh --zookeeper $zks --create --topic $2 --config message.timestamp.type=LogAppendTime --replication-factor $3 --partitions $4"
		kafka-topics.sh --zookeeper $zks --create --topic $2 --config message.timestamp.type=LogAppendTime --replication-factor $3 --partitions $4
        ;;
    delete)
        echo "kafka-topics.sh --zookeeper $zks --delete --topic $2"
		kafka-topics.sh --zookeeper $zks --delete --topic $2
        ;;
    produce)
        echo "kafka-console-producer.sh --broker-list $brks --topic $2"
		kafka-console-producer.sh --broker-list $brks --topic $2
        ;;
    consume)
        echo "kafka-console-consumer.sh --zookeeper $zks --from-beginning --topic $2"
		kafka-console-consumer.sh --zookeeper $zks --from-beginning --topic $2
        ;;
    group_consume)
        echo "kafka-console-consumer.sh --zookeeper $zks --topic $2"
		kafka-console-consumer.sh --zookeeper $zks --from-beginning --topic $2 --consumer.config config/consumer.properties
        ;;
    delete_group)
        echo "kafka-consumer-groups.sh --zookeeper $zks --delete --group $2"
              kafka-consumer-groups.sh --zookeeper $zks --delete --group $2
        ;;
    delete_topic_from_group)
        echo "kafka-consumer-groups.sh --zookeeper $zks --delete --group $2 --topic $3"
              kafka-consumer-groups.sh --zookeeper $zks --delete --group $2 --topic $3
        ;;
    desc)
        echo "kafka-topics.sh --zookeeper $zks --describe --topic $2"
		kafka-topics.sh --zookeeper $zks --describe --topic $2
        echo "largest: kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list $brks --topic $2 --time -1"
                kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list $brks --topic $2 --time -1
        echo "earliest: kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list $brks --topic $2 --time -2"
                kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list $brks --topic $2 --time -2
        ;;
    produce_test)
       echo "kafka-producer-perf-test.sh --topic $2 --throughput -1 --record-size 10 --num-records $3 --producer-props bootstrap.servers=$brks"
             kafka-producer-perf-test.sh --topic $2 --throughput -1 --record-size 10 --num-records $3 --producer-props bootstrap.servers=$brks
        ;;
    *)
       echo "kafka (list | create | delete | produce | produce_test | consume | group_consume | delete_group | delete_topic_from_group  | desc | *)"
esac

cd $old

exit 0
-----------------------------------------------------------
```

* kafka启动脚本
```aidl
vi bin/kafkaStart
-----------------------------------------------------------
#!/bin/bash

curr_dir=`pwd`

KAFKA_HOME=/opt/softwares/kafka_2.11-0.11.0.0
ssh admin@hadoop01 "$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties > /dev/null 2>&1 &"
ssh admin@hadoop02 "$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties > /dev/null 2>&1 &"
ssh admin@hadoop03 "$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties > /dev/null 2>&1 &"

xcall jps

cd $curr_dir
-----------------------------------------------------------
```

* kafka停机脚本
```aidl
vi bin/kafkaStop
-----------------------------------------------------------
#!/bin/bash

curr_dir=`pwd`

KAFKA_HOME=/opt/softwares/kafka_2.11-0.11.0.0

ssh admin@hadoop01 "$KAFKA_HOME/bin/kafka-server-stop.sh stop > /dev/null 2>&1 &"
ssh admin@hadoop02 "$KAFKA_HOME/bin/kafka-server-stop.sh stop > /dev/null 2>&1 &"
ssh admin@hadoop03 "$KAFKA_HOME/bin/kafka-server-stop.sh stop > /dev/null 2>&1 &"

xcall jps

cd $curr_dir
-----------------------------------------------------------
```

## mysql binlog配置
```aidl
vi /etc/my.cnf
-----------------------------------------------------------

server-id= 1
log-bin=mysql-bin
# full 全量字段,minimal 只有改变字段
binlog_row_image=full
binlog_format=row
binlog-do-db=test
-----------------------------------------------------------

# 重启
sudo service mysqld restart

# 授权
CREATE USER 'debezium'@'%' IDENTIFIED BY 'debezium';
GRANT ALL PRIVILEGES ON  *.* to "debezium"@"%" IDENTIFIED BY "debezium";
FLUSH PRIVILEGES;
```

## 启动测试
```aidl
zkStart

kafkaStart

kafkaConnectorStart
```

* 新增topic
```aidl
kafka list | grep connect
connect-configs
connect-offsets
connect-status
```

## 注册同步链路
```aidl
# 请求
curl -H "Content-Type: application/json" -X POST -d  '{
      "name" : "inventory-connector",
      "config" : {
          "connector.class" : "io.debezium.connector.mysql.MySqlConnector",
         "database.hostname" : "hadoop01",
          "database.port" : "3306",
          "database.user" : "debezium",
          "database.password" : "debezium",
          "database.server.id" : "1",
          "database.server.name" : "debezium",
          "database.whitelist" : "debezium_test",
          "database.history.kafka.bootstrap.servers":"hadoop01:9092,hadoop02:9092,hadoop03:9092",
          "database.history.kafka.topic":"debezium_binlog",
          "include.schema.change":"true"
      }
  }' http://hadoop01:8083/connectors
  
注：
1.database.server.name 是数据库对应逻辑名称，database.whitelist 是需要同步的库名(与canal、maxwell不同，无需在my.cnf中注册需要同步的表，此处指定谁，就可主动抓取谁的binlog);
2.详细配置参考 https://debezium.io/documentation/reference/1.2/connectors/mysql.html 。
  
# 响应
{"name":"inventory-connector","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","database.hostname":"hadoop01","database.port":"3306","database.user":"debezium","database.password":"debezium","database.server.id":"1","database.server.name":"debezium","database.whitelist":"debezium_test","database.history.kafka.bootstrap.servers":"hadoop01:9092,hadoop02:9092,hadoop03:9092","database.history.kafka.topic":"debezium_binlog","include.schema.change":"true","name":"inventory-connector"},"tasks":[{"connector":"inventory-connector","task":0}],"type":"source"}```

# 查看同步配置是否生效
curl -XGET  http://hadoop01:8083/connectors
["inventory-connector"]

# 查看同步配置信息
curl -XGET  http://hadoop01:8083/connectors/inventory-connector

# 查看同步配置运行状态
curl -XGET  http://hadoop01:8083/connectors/inventory-connector/status
响应
{"name":"inventory-connector","connector":{"state":"RUNNING","worker_id":"192.168.152.103:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"192.168.152.103:8083"}],"type":"source"}[admin@hadoop01 ~]$

# 查看 新增topic
kafka list
debezium  << database.server.name
debezium_binlog << database.history.kafka.topic
```

## 演示binlog
```aidl
use debezium_test;

create table student(id int(3) primary key auto_increment,name varchar(50),score int(3));

insert into student(name,score) values('a1',98);

```
* 找到最新binlog文件（默认在 /var/lib/mysql/ 目录）
```aidl
mysql> show binary logs;
+------------------+------------+
| Log_name         | File_size  |
+------------------+------------+
| mysql-bin.000001 |        732 |
| mysql-bin.000002 |        177 |
| mysql-bin.000003 |   80873886 |
| mysql-bin.000004 | 1080773872 |
| mysql-bin.000005 |   96386423 |
| mysql-bin.000006 |    3051102 |
| mysql-bin.000007 |  214750185 |
| mysql-bin.000008 |        969 |
| mysql-bin.000009 |        177 |
| mysql-bin.000010 | 1102045188 |
| mysql-bin.000011 | 1116717194 |
| mysql-bin.000012 |  793675797 |
| mysql-bin.000013 |       3041 |
| mysql-bin.000014 |        177 |
| mysql-bin.000015 |        177 |
| mysql-bin.000016 |        177 |
| mysql-bin.000017 |        177 |
| mysql-bin.000018 |        401 |
| mysql-bin.000019 |        177 |
| mysql-bin.000020 |        177 |
| mysql-bin.000021 |        177 |
| mysql-bin.000022 |        177 |
| mysql-bin.000023 |        177 |
| mysql-bin.000024 |       2079 |
| mysql-bin.000025 |        177 |
| mysql-bin.000026 |        399 | <<<<
+------------------+------------+
```

* 查找当前正在写入binlog文件
```aidl
mysql> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000026  <<<
         Position: 686
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

ERROR:
No query specified
```

* 查找mysql-bin.000026中binlog事件
```aidl
mysql> show binlog events in 'mysql-bin.000026';
+------------------+-----+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                                                                          |
+------------------+-----+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------+
| mysql-bin.000026 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.18-log, Binlog ver: 4                                                                         |
| mysql-bin.000026 | 123 | Previous_gtids |         1 |         154 |                                                                                                               |
| mysql-bin.000026 | 154 | Anonymous_Gtid |         1 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                          |
| mysql-bin.000026 | 219 | Query          |         1 |         399 | use `debezium_test`; create table student(id int(3) primary key auto_increment,name varchar(50),score int(3)) |
| mysql-bin.000026 | 399 | Anonymous_Gtid |         1 |         464 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                          |
| mysql-bin.000026 | 464 | Query          |         1 |         545 | BEGIN                                                                                                         |
| mysql-bin.000026 | 545 | Table_map      |         1 |         608 | table_id: 881 (debezium_test.student)                                                                         |
| mysql-bin.000026 | 608 | Write_rows     |         1 |         655 | table_id: 881 flags: STMT_END_F                                                                               | <<< 写入数据
| mysql-bin.000026 | 655 | Xid            |         1 |         686 | COMMIT /* xid=649 */                                                                                          |
+------------------+-----+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------+

```

* 查看binlog
```aidl
sudo mysqlbinlog --no-defaults -v -v --base64-output=DECODE-ROWS /var/lib/mysql/mysql-bin.000026 | tail -n 20

SET TIMESTAMP=1622268842/*!*/;
BEGIN
/*!*/;
# at 545
#210529 14:14:02 server id 1  end_log_pos 608 CRC32 0xd1a01157 	Table_map: `debezium_test`.`student` mapped to number 881
# at 608
#210529 14:14:02 server id 1  end_log_pos 655 CRC32 0xa82e02e2 	Write_rows: table id 881 flags: STMT_END_F
### INSERT INTO `debezium_test`.`student`   <<<<<
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='a1' /* VARSTRING(50) meta=50 nullable=1 is_null=0 */
###   @3=98 /* INT meta=0 nullable=1 is_null=0 */
# at 655   <<<<<
#210529 14:14:02 server id 1  end_log_pos 686 CRC32 0x7423ae77 	Xid = 649
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

* 修改记录
```aidl
update student set score=50 where id=1;
```

* full 风格binlog(所有列都带上)
```aidl
sudo mysqlbinlog --no-defaults -v -v --base64-output=DECODE-ROWS /var/lib/mysql/mysql-bin.000026 | tail -n 20

#210529 14:26:18 server id 1  end_log_pos 895 CRC32 0xf1883ab9 	Table_map: `debezium_test`.`student` mapped to number 881
# at 895
#210529 14:26:18 server id 1  end_log_pos 955 CRC32 0x91434f23 	Update_rows: table id 881 flags: STMT_END_F
### UPDATE `debezium_test`.`student`  <<<<<
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='a1' /* VARSTRING(50) meta=50 nullable=1 is_null=0 */
###   @3=98 /* INT meta=0 nullable=1 is_null=0 */
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='a1' /* VARSTRING(50) meta=50 nullable=1 is_null=0 */
###   @3=50 /* INT meta=0 nullable=1 is_null=0 */
# at 955 <<<<<
#210529 14:26:18 server id 1  end_log_pos 986 CRC32 0xaf5c23b5 	Xid = 653
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

* minimal 风格binlog（只有被修改列被带上）
```aidl
set  binlog_row_image ='minimal';

update student set score=55 where id=1;

sudo mysqlbinlog --no-defaults -v -v --base64-output=DECODE-ROWS /var/lib/mysql/mysql-bin.000026 | tail -n 20

SET TIMESTAMP=1622269694/*!*/;
BEGIN
/*!*/;
# at 1132
#210529 14:28:14 server id 1  end_log_pos 1195 CRC32 0x70a769fd 	Table_map: `debezium_test`.`student` mapped to number 881
# at 1195
#210529 14:28:14 server id 1  end_log_pos 1241 CRC32 0x79a7963b 	Update_rows: table id 881 flags: STMT_END_F
### UPDATE `debezium_test`.`student`  <<<<<
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
### SET
###   @3=55 /* INT meta=0 nullable=1 is_null=0 */
# at 1241  <<<<<
#210529 14:28:14 server id 1  end_log_pos 1272 CRC32 0x9d7fa34a 	Xid = 655
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;

```

## 新增 topic
```aidl
kafka list
debezium.debezium_test.student
```

## kafka 中数据
```aidl
kafka consume debezium.debezium_test.student
既有schema 又有数据，非常重
{
	"schema":{
		"type":"struct",
		"fields":[
			{
				"type":"struct",
				"fields":[
					{
						"type":"int32",
						"optional":false,
						"field":"id"
					},
					{
						"type":"string",
						"optional":true,
						"field":"name"
					},
					{
						"type":"int32",
						"optional":true,
						"field":"score"
					}
				],
				"optional":true,
				"name":"debezium.debezium_test.student.Value",
				"field":"before"
			},
			{
				"type":"struct",
				"fields":[
					{
						"type":"int32",
						"optional":false,
						"field":"id"
					},
					{
						"type":"string",
						"optional":true,
						"field":"name"
					},
					{
						"type":"int32",
						"optional":true,
						"field":"score"
					}
				],
				"optional":true,
				"name":"debezium.debezium_test.student.Value",
				"field":"after"
			},
			{
				"type":"struct",
				"fields":[
					{
						"type":"string",
						"optional":false,
						"field":"version"
					},
					{
						"type":"string",
						"optional":false,
						"field":"connector"
					},
					{
						"type":"string",
						"optional":false,
						"field":"name"
					},
					{
						"type":"int64",
						"optional":false,
						"field":"ts_ms"
					},
					{
						"type":"string",
						"optional":true,
						"name":"io.debezium.data.Enum",
						"version":1,
						"parameters":{
							"allowed":"true,last,false"
						},
						"default":"false",
						"field":"snapshot"
					},
					{
						"type":"string",
						"optional":false,
						"field":"db"
					},
					{
						"type":"string",
						"optional":true,
						"field":"table"
					},
					{
						"type":"int64",
						"optional":false,
						"field":"server_id"
					},
					{
						"type":"string",
						"optional":true,
						"field":"gtid"
					},
					{
						"type":"string",
						"optional":false,
						"field":"file"
					},
					{
						"type":"int64",
						"optional":false,
						"field":"pos"
					},
					{
						"type":"int32",
						"optional":false,
						"field":"row"
					},
					{
						"type":"int64",
						"optional":true,
						"field":"thread"
					},
					{
						"type":"string",
						"optional":true,
						"field":"query"
					}
				],
				"optional":false,
				"name":"io.debezium.connector.mysql.Source",
				"field":"source"
			},
			{
				"type":"string",
				"optional":false,
				"field":"op"
			},
			{
				"type":"int64",
				"optional":true,
				"field":"ts_ms"
			},
			{
				"type":"struct",
				"fields":[
					{
						"type":"string",
						"optional":false,
						"field":"id"
					},
					{
						"type":"int64",
						"optional":false,
						"field":"total_order"
					},
					{
						"type":"int64",
						"optional":false,
						"field":"data_collection_order"
					}
				],
				"optional":true,
				"field":"transaction"
			}
		],
		"optional":false,
		"name":"debezium.debezium_test.student.Envelope"
	},
	"payload":{
		"before":null,
		"after":{
			"id":1,
			"name":"a1",
			"score":98
		},
		"source":{
			"version":"1.2.0.Final",
			"connector":"mysql",
			"name":"debezium",
			"ts_ms":1622268842000,
			"snapshot":"false",
			"db":"debezium_test",
			"table":"student",
			"server_id":1,
			"gtid":null,
			"file":"mysql-bin.000026",
			"pos":608,
			"row":0,
			"thread":9,
			"query":null
		},
		"op":"c",
		"ts_ms":1622268842853,
		"transaction":null
	}
}
```

## 删除同步链路
```aidl
curl -X DELETE http://hadoop01:8083/connectors/inventory-connector
```