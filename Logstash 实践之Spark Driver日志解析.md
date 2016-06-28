# Logstash 实践之Spark Driver日志解析

## logstash config script
```ruby
input {
    file {
        path => ["/apps/svr/logstash/log/telescope.log"]
        start_position => "beginning"
        type => "mysql_slowlog"
    }
}

filter {
    multiline {
      pattern => "(^\d+\sERROR)|(^.+Exception:.+)|(^\s+at .+)|(^\s+... \d+ more)|(^\s*Causedby:.+)"
      what => "previous"
    }
    grok {
        match => ["message", "(?<log_time>%{YEAR}[/-]%{MONTHNUM}[/-]%{MONTHDAY}\s+%{TIME}?)\s+(?<log_level>\w+)\s+\[(?<class_name>.+?)\]: %{GREEDYDATA:message}"]
        overwrite => ["message"]
    }

    ruby {
      code => " event['job_name']=event['path'].split('/')[-1].gsub('-', '_').downcase"
    }
}
output {
    stdout {
        codec => rubydebug
    }

    elasticsearch {
        host => 'localhost'
        protocol => 'transport'
        cluster => 'elasticsearch'
        index => 'logstash-spark-driver-%{+YYYY.MM.dd}'
    }
}
```
Sample spark driver log

```
15/12/07 23:34:21 WARN [org.apache.spark.SparkConf---main]: The configuration key 'spark.shuffle.file.buffer.kb' has been deprecated as of Spark 1.4 and and may be removed in the future. Please use the new key 'spark.shuffle.file.buffer' instead.
15/12/07 23:34:21 INFO [org.apache.spark.SparkContext---main]: Running Spark version 1.5.2
15/12/07 23:34:21 WARN [org.apache.hadoop.util.NativeCodeLoader---main]: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
15/12/07 23:34:21 WARN [org.apache.spark.SparkConf---main]: The configuration key 'spark.shuffle.file.buffer.kb' has been deprecated as of Spark 1.4 and and may be removed in the future. Please use the new key 'spark.shuffle.file.buffer' instead.
15/12/07 23:34:21 WARN [org.apache.spark.SparkConf---main]: In Spark 1.0 and later spark.local.dir will be overridden by the value set by the cluster manager (via SPARK_LOCAL_DIRS in mesos/standalone and LOCAL_DIRS in YARN).
15/12/07 23:34:21 INFO [org.apache.spark.SecurityManager---main]: Changing view acls to: spark
15/12/07 23:34:21 INFO [org.apache.spark.SecurityManager---main]: Changing modify acls to: spark
15/12/07 23:34:21 INFO [org.apache.spark.SecurityManager---main]: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(spark); users with modify permissions: Set(spark)
15/12/07 23:34:22 INFO [akka.event.slf4j.Slf4jLogger---sparkDriver-akka.actor.default-dispatcher-3]: Slf4jLogger started
```
Output:
```xml
{
       "message" => "Added rdd_1772_6 in memory on 10.201.113.173:22750 (size: 931.0 KB, free: 3.1 GB)",
      "@version" => "1",
    "@timestamp" => "2015-12-23T07:42:16.249Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/telescope.log",
          "type" => "mysql_slowlog",
      "log_time" => "15/12/07 23:45:40",
     "log_level" => "INFO",
    "class_name" => "org.apache.spark.storage.BlockManagerInfo---sparkDriver-akka.actor.default-dispatcher-15",
      "job_name" => "telescope.log"
}
{
       "message" => "Finished task 6.0 in stage 145.0 (TID 2010) in 218 ms on 10.201.113.173 (6/9)",
      "@version" => "1",
    "@timestamp" => "2015-12-23T07:42:16.249Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/telescope.log",
          "type" => "mysql_slowlog",
      "log_time" => "15/12/07 23:45:40",
     "log_level" => "INFO",
    "class_name" => "org.apache.spark.scheduler.TaskSetManager---task-result-getter-0",
      "job_name" => "telescope.log"
}
```

Kibana 界面：
![这里写图片描述](http://img.blog.csdn.net/20160131232559191)