# 用sqoop2从mysql导入数据到hbase

标签（空格分隔）： sqoop2 hbase hadoop mysql 数据迁移

---

##**一、基础环境**
    hadoop-2.6.0
    sqoop-1.99.6
    hbase-1.0.1.1
##**二、数据迁移实际操作步骤**
###**1、mysql中表结构显示**

    CREATE TABLE `risk_trade_error_result` (
    `SERIALID` bigint(11) NOT NULL AUTO_INCREMENT,
    `MERCHANT_NO` varchar(32) NOT NULL,
    `TRADE_TIME` varchar(19) DEFAULT NULL,
    `TRADE_TYPE` char(12) DEFAULT NULL COMMENT '交易类型',
    `TERMINAL_NO` varchar(64) NOT NULL,
    `CARDNO` varchar(20) NOT NULL COMMENT '卡号',
    `AMOUNT` bigint(12) DEFAULT NULL COMMENT '交易金额',
    `RISK_CODE` varchar(6) DEFAULT NULL,
    `POS_CODE` varchar(2) DEFAULT NULL,
    `REQUESTID` varchar(64) DEFAULT NULL,
    `RISK_LEVEL` varchar(4) DEFAULT NULL,
    PRIMARY KEY (`SERIALID`),
    KEY `idx_merchant_no` (`MERCHANT_NO`),
    KEY `idx_terminal_no` (`TERMINAL_NO`),
    KEY `idx_trade_time` (`TRADE_TIME`)
    ) ENGINE=InnoDB AUTO_INCREMENT=3146182 DEFAULT CHARSET=utf8

注意：
mysql中的字段名最好用大写！
###**2、使用Phoenix创建hbase表**
    //进入Phoenix shell模式，命令如下：
    #/home/hadoop/phoenix-4.5.0-HBase-1.0-bin/bin/sqlline.py risk1（主机名）  
    //创建hbase表：
    create table RISK.RISK_TRADE_ERROR_RESULT(
    TRADE_TIME UNSIGNED_LONG not null,
    MERCHANT_NO varchar(32) not null,
    TRADE_TYPE char(12),
    TERMINAL_NO varchar(64) not null,
    CARDNO varchar(20),
    AMOUNT bigint(12),
    RISK_CODE varchar(6),
    POS_CODE varchar(2) not null,
    REQUESTID varchar(64),
    RISK_LEVEL varchar(4),
    SERIALID bigint(11) not null,
    CONSTRAINT PK PRIMARY KEY (TRADE_TIME,MERCHANT_NO,TERMINAL_NO,POS_CODE,SERIALID)
    ) ;
    

注意：
表的字段名一定要跟mysql中的字段一致，数据类型可以不一致！

###**3、启动sqoop2-server**
    #/home/hadoop/sqoop-1.99.6-bin-hadoop200/bin/sqoop2-server start
    如果遇到疑似server端的问题，可以先stop，然后再start，再重试。
###**4、进入客户端shell**
    #/home/hadoop/sqoop-1.99.6-bin-hadoop200/bin/sqoop2-shell
###**5、为客户端配置服务器**
    sqoop:000>set server --host risk1 --port 12000 --webapp sqoop
###**6、根据connector创建mysql link**
    sqoop:000>show link
    +----+------------------------+---------+--------------------------    ----------------------------+----------------------+
    | Id |          Name          | Version |                        Class                         | Supported Directions |
    +----+------------------------+---------+------------------------------------------------------+----------------------+
    | 1  | kite-connector         | Unknown | org.apache.sqoop.connector.kite.KiteConnector        | FROM/TO              |
    | 2  | kafka-connector        | Unknown | org.apache.sqoop.connector.kafka.KafkaConnector      | TO                   |
    | 3  | hdfs-connector         | Unknown | org.apache.sqoop.connector.hdfs.HdfsConnector        | FROM/TO              |
    | 4  | generic-jdbc-connector | Unknown | org.apache.sqoop.connector.jdbc.GenericJdbcConnector | FROM/TO              |
    +----+------------------------+---------+------------------------------------------------------+----------------------+
    
    sqoop:000>create link --c 4
    //根据如下信息输入mysql相关内容，password无需输入
    link with id 22 and name mysql_link (Enabled: true, Created by root at 16-4-14 下午7:49, Updated by root at 16-4-14 下午7:55)
    Using Connector generic-jdbc-connector with id 4
    Link configuration
    JDBC Driver Class: com.mysql.jdbc.Driver
    JDBC Connection String: jdbc:mysql://172.20.4.110:5306/riskxn
    Username: riskwang
    Password: 
    JDBC Connection Properties: 

###**7、根据connector创建hbase link**
    sqoop:000>create link --c 4
    //根据如下信息输入phoenix SQL on hbase相关内容，Username和password都无需输入
    link with id 23 and name hbase_link (Enabled: true, Created by root at 16-4-15 上午10:10, Updated by root at 16-4-15 上午10:10)
    Using Connector generic-jdbc-connector with id 4
    Link configuration
    JDBC Driver Class: org.apache.phoenix.jdbc.PhoenixDriver
    JDBC Connection String: jdbc:phoenix:172.20.4.113
    Username: 
    Password: 
    JDBC Connection Properties: 

###**8、根据link创建job**
    sqoop:000>show link
    +----+--------------------+--------------+------------------------+---------+
    | Id |        Name        | Connector Id |     Connector Name     | Enabled |
    +----+--------------------+--------------+------------------------+---------+
    | 21 | hdfs_link          | 3            | hdfs-connector         | true    |
    | 22 | mysql_link         | 4            | generic-jdbc-connector | true    |
    | 23 | hbase_link         | 4            | generic-jdbc-connector | true    |
    +----+--------------------+--------------+------------------------+---------+
    
    sqoop:000>create job --f 22 --t 23
    //根据如下内容输入相关的job信息
    1 job(s) to show: 
    Job with id 99 and name trade_mysql_to_hbase (Enabled: true, Created by root at 16-4-18 上午10:19, Updated by root at 16-4-19 上午11:45)
    Throttling resources
    Extractors: 3
    Loaders: 2
    From link: 22
    From database configuration
    Schema name: 
    Table name: 
    Table SQL statement: select SERIALID,MERCHANT_NO,UNIX_TIMESTAMP(TRADE_TIME) TRADE_TIME,TRADE_TYPE,TERMINAL_NO,CARDNO,AMOUNT,RISK_CODE,POS_CODE,REQUESTID,RISK_LEVEL from riskxn.risk_trade_error_result where ${CONDITIONS} and left(FROM_UNIXTIME(UNIX_TIMESTAMP(TRADE_TIME)),10)=DATE_SUB(curdate(), INTERVAL 1 DAY)
    Table column names: 
    Partition column name: SERIALID
    Null value allowed for the partition column: 
    Boundary query: 
    Incremental read
    Check column: 
    Last value: 
    To link: 23
    To database configuration
    Schema name: @RISK
    Table name: RISK_TRADE_ERROR_RESULT
    Table SQL statement: 
    Table column names: 
    Stage table name: 
    Should clear stage table: 

注意：
目前增量导入由以上SQL语句实现，Incremental read部分的增量导入，sqoop-1.99.6版本还有问题，目前无法实现。

###**9、开始运行job**
    sqoop:000>start job --j 99     //99是上一步创建job时的ID
###**10、脚本运行**
    #vi script.sqoop
    ####输入如下内容
    set server --host risk1 --port 12000 --webapp sqoop
    set option --name verbose --value true
    start job --jid 99
###**11、创建Linux crontab定时任务，实现每天自动定时导入数据到hbase**
    #crontab –e
    ###输入如下内容，每天中午12点会定时执行脚本script.sqoop，将mysql中的数据导到hbase
    00 12 * * * . /etc/profile;/bin/sh /home/hadoop/sqoop-1.99.6-bin-hadoop200/bin/sqoop.sh client /home/hadoop/sqoop-1.99.6-bin-hadoop200/script.sqoop


##**三、问题总结**
###3.1、mysql中的字段名要跟hbase表的字段名一直。
    mysql表中的字段需要大写，因为Phoenix建表时会自动将字段转换为大写。
###3.2、错误提示如下
	Exception has occurred during processing command
    Exception: org.apache.sqoop.common.SqoopException Message: CLIENT_0001:Server has returned exception
修改方法：
set option --name verbose --value true
###3.3、增量导入问题，错误提示如下
	Error message: Size of input exceeds allowance for this input field. Maximal allowed size is -1
修改方法：
如果后面用到，需要修改sqoop2源码，目前使用创建job时的SQL语句控制增量导入到hbase，不需要修改，如果后面需要改这个问题，请参考：
http://blog.csdn.net/xrf416933696/article/details/50171777

###3.4、其他问题主要是：
    Mysql语句的问题和Phoenix建表时的问题，注意一下就可以避免。
    另外要保证hdfs、yarn、historyserver正常启动！！！


