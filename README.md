**Configuration to log ranger audit to a kafka topic using kafka log4j appender**

Here is the config I have tried in knox to implement log4j appender and write to kafka topic  in a kerberos environment

    ====  
    log4j.logger.org.apache.ranger.audit.provider.Log4jAuditProvider=INFO,KAFKA_knox_AUDIT  
    log4j.appender.KAFKA_knox_AUDIT=org.apache.kafka.log4jappender.KafkaLog4jAppender  
    log4j.appender.KAFKA_knox_AUDIT.brokerList=c416-node2.raghav.com:6667  
    log4j.appender.KAFKA_knox_AUDIT.topic=knox_audit_log  
    log4j.appender.KAFKA_knox_AUDIT.securityProtocol=SASL_PLAINTEXT  
    log4j.appender.KAFKA_knox_AUDIT.saslKerberosServiceName=kafka  
    log4j.appender.KAFKA_knox_AUDIT.clientJaasConfPath=<KafkaclientJass>  
    log4j.appender.KAFKA_knox_AUDIT.layout=org.apache.log4j.PatternLayout  
    log4j.appender.KAFKA_knox_AUDIT.layout.ConversionPattern=%d{ISO8601} %-5p [%t]: %c{2} (%F:%M(%L)) - %m%n  
    log4j.appender.KAFKA_knox_AUDIT.ProducerType=async  
    === 

 
  
Additionally below two properties are required to enable log4j audit (these two properties are added in custom ranger-knox-audit  
  
    ====  
    <property>  
    <name>xasecure.audit.destination.log4j</name>  
    <value>true</value>  
    </property>  
      
    <property>  
    <name>xasecure.audit.destination.log4j.logger</name>  
    <value>org.apache.ranger.audit.provider.Log4jAuditProvider</value>  
    </property>  
    ===  

  
-->Required jars copied under /usr/hdp/current/knox-server/dep  

    ===  
    kafka-log4j-appender-1.0.0.2.6.5.0-292.jar  
    kafka_2.11-1.0.0.2.6.5.0-292.jar  
    kafka-clients-1.0.0.2.6.5.0-292.jar  
    metrics-core-2.2.0.jar  
    scala-library-2.11.12.jar  
    ===  

  
-->Topic created on kafka with all privileges :  
  

    #/usr/hdp/current/kafka-broker/bin/kafka-topics.sh --zookeeper <zookeeperHostnae>:2181 --create --topic knox_audit_log --partitions 1 --replication-factor 1  
      

--> Added ACL to all users to write to the topic:  
  

    #/usr/hdp/current/kafka-broker/bin/kafka-acls.sh --topic knox_audit_log --allow-principal User:* --operation All --authorizer-properties zookeeper.connect=c416-node2.raghav.com:2181 -add  
      

  
-->Restart and try access knox with ranger plugin enabled.  
  
Monitor message being written to kafka topic with the console-consumer : (sample message output below)  
  

    #/usr/hdp/current/kafka-broker/bin/kafka-console-consumer.sh --zookeeper <zookeeper> --topic knox_audit_log --from-beginning --security-protocol PLAINTEXTSASL  
    [..]  
    2018-11-08 01:51:08,003 INFO [org.apache.ranger.audit.queue.AuditBatchQueue1]: provider.Log4jAuditProvider (Log4JAuditDestination.java:logJSON(107)) - {"repoType":5,"repo":"c416_knox","reqUser":"admin","evtTime":"2018-11-08 01:51:07.437","access":"allow","resource":"default/WEBHDFS","resType":"service","action":"allow","result":1,"policy":19,"enforcer":"ranger-acl","cliIP":"172.25.39.129","agentHost":"c416-node2.raghav.com","logType":"RangerAudit","id":"d3fd4eb9-56bb-44f5-9594-435d4afe79a9-2","seq_num":5,"event_count":1,"event_dur_ms":0,"tags":[],"cluster_name":"c416"}  
      
    ^CProcessed a total of 2 messages  
      

 
-->Execute curl command to access knox which is logged to kafka topic as above:  
  

    #curl -u admin:admin-password 'https://c416-node2.raghav.com:8443/gateway/default/webhdfs/v1/tmp/?op=liststatus'  

  As similar config can be done with other services log4j to achieve same.

Audit messages can be further customized based on requirement. By default log4j appender writes audit in json format. Format of the audit  can be changed by setting the ConversionPattern

For example setting ConversionPattern will audit messages to kafka with  json format line 

    log4j.appender.KAFKA_knox_AUDIT.layout.ConversionPattern= %m%n  

Like below: 

    {"repoType":5,"repo":"c416_knox","reqUser":"admin","evtTime":"2018-11-08 02:19:13.240","access":"allow","resource":"default/WEBHDFS","resType":"service","action":"allow","result":1,"policy":19,"enforcer":"ranger-acl","cliIP":"172.25.39.129","agentHost":"c416-node2.raghav.com","logType":"RangerAudit","id":"543e44f5-2ec9-468a-800a-22359c333c11-0","seq_num":1,"event_count":1,"event_dur_ms":0,"tags":[],"cluster_name":"c416"}



--> Hive log4j2 config for audit to log4j : 

Update 'hive-log4j2' with below configs :
-->Add list new appender to appenders: (added fileaudit here)
```
# list of all appenders
appenders = console, DRFA, fileaudit
```

-->Add xaaudit to list of loggers : (Added xaaudit)
```
# list of all loggers
loggers = NIOServerCnxn, ClientCnxnSocketNIO, DataNucleus, Datastore, JPOX, xaaudit
```

-->Add Appender definition (here using RollingFile type)
```
# daily rolling file appender
appender.fileaudit.type = RollingFile
appender.fileaudit.name = fileaudit
appender.fileaudit.fileName = /tmp/audit_test_hive.log
appender.fileaudit.filePattern = /tmp/audit_test_hive.log.%d{yyyy-MM-dd}_%i.gz
appender.fileaudit.layout.type = PatternLayout
appender.fileaudit.layout.pattern = %d{ISO8601} %-5p [%t]: %c{2} (%F:%M(%L)) - %m%n
appender.fileaudit.policies.type = Policies
appender.fileaudit.policies.time.type = TimeBasedTriggeringPolicy
appender.fileaudit.policies.time.interval = 1
appender.fileaudit.policies.time.modulate = true
appender.fileaudit.strategy.type = DefaultRolloverStrategy
appender.fileaudit.strategy.max = {{hive2_log_maxbackupindex}}
appender.fileaudit.policies.fsize.type = SizeBasedTriggeringPolicy
appender.fileaudit.policies.fsize.size = {{hive2_log_maxfilesize}}MB
```

-->Add Logger definition
```
logger.xaaudit.name = xaaudit
logger.xaaudit.level = INFO
logger.xaaudit.appenderRef.fileaudit.ref = fileaudit
```

First two steps is updating the existing properties (appenders and loggers). Next two configs should be added in the end of hive-log4j2

-->Added below to configs under custom ranger-hive-audit :
```
xasecure.audit.destination.log4j=true
xasecure.audit.destination.log4j.logger=xaaudit
```
Restart HS2 and verify the log file /tmp/audit_test_hive.log after executing some hive queries.
