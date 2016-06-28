# Logstash实践之MySQL Slowlog解析
## logstash config script
```ruby
input {
    file {
        path => ["/apps/svr/logstash/log/slow3306.log3"]
        start_position => "beginning"
        type => "mysql_slowlog"
    }
}

filter {

  # I am told that the '# Time: ...' lines in slow query log
  # are optional and may not appear, so merge it to the next line.
  multiline {
    what => next
    pattern => "^# Time:"
  }
  # The next line is always the '# user@host ...' line, so merge
  # everything that is not that upwards towards it.
  multiline {
    what => previous
    negate => true
    pattern => "^# [A-Za-z0-9_-]+@"
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
        index => 'logstash-mysql-slowlog-%{+YYYY.MM.dd}'
    }
}
```
## Output
```json
{
       "message" => "# Time: 151217  0:06:07\n# User@Host: vipshop_dba[vipshop_dba] @  [xxx.xxx.xxx.xxx]\n# Query_time: 0.172298  Lock_time: 0.000337 Rows_sent: 0  Rows_examined: 38819\nSET timestamp=1450281967;\nselect t1.table_name,t2.column_name,\"null\" as extra from information_schema.tables t1 left join information_schema.columns t2 on t1.table_name=t2.table_name and (t2.column_name =\"create_time\" or t2.column_name =\"test_create_time\") where t1.table_schema=\"schemareview_1450281966\" and t2.column_name is null union select t1.table_name,t2.column_name,\"null\" as extra from information_schema.tables t1 left join information_schema.columns t2 on t1.table_name=t2.table_name and (t2.column_name =\"update_time\" or t2.column_name =\"test_update_time\") where t1.table_schema=\"schemareview_1450281966\" and t2.column_name is null;",
      "@version" => "1",
    "@timestamp" => "2015-12-23T05:53:18.527Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/slow3306.log2",
          "type" => "mysql_slowlog",
          "tags" => [
        [0] "multiline"
    ]
}
{
       "message" => "# Time: 151217  0:06:16\n# User@Host: vipshop_dba[vipshop_dba] @  [xx.xxx.xxx.xxx]\n# Query_time: 0.104545  Lock_time: 0.000295 Rows_sent: 0  Rows_examined: 7761\nuse schemareview_1450281976;\nSET timestamp=1450281976;\nselect t.table_name,c.column_name,\"\" as extra from information_schema.tables t left join information_schema.columns c on t.table_name=c.table_name and t.table_schema=c.table_schema and column_key=\"pri\" where t.table_schema=\"schemareview_1450281976\" and c.column_name is null;",
      "@version" => "1",
    "@timestamp" => "2015-12-23T05:53:18.533Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/slow3306.log2",
          "type" => "mysql_slowlog",
          "tags" => [
        [0] "multiline"
    ]
}
{
       "message" => "# User@Host: vipshop_dba[vipshop_dba] @  [xxx.xxx.xxx.xxx]\n# Query_time: 0.171176  Lock_time: 0.001116 Rows_sent: 0  Rows_examined: 38819\nuse vipshop_dba;\nSET timestamp=1450281976;\nselect t1.table_name,t2.column_name,\"null\" as extra from information_schema.tables t1 left join information_schema.columns t2 on t1.table_name=t2.table_name and (t2.column_name =\"create_time\" or t2.column_name =\"test_create_time\") where t1.table_schema=\"schemareview_1450281976\" and t2.column_name is null union select t1.table_name,t2.column_name,\"null\" as extra from information_schema.tables t1 left join information_schema.columns t2 on t1.table_name=t2.table_name and (t2.column_name =\"update_time\" or t2.column_name =\"test_update_time\") where t1.table_schema=\"schemareview_1450281976\" and t2.column_name is null;",
      "@version" => "1",
    "@timestamp" => "2015-12-23T05:53:18.536Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/slow3306.log2",
          "type" => "mysql_slowlog",
          "tags" => [
        [0] "multiline"
    ]
}
{
       "message" => "# Time: 151217  0:06:23\n# User@Host: vipshop_dba[vipshop_dba] @  [xx.xx.xx.xxx]\n# Query_time: 0.185655  Lock_time: 0.000390 Rows_sent: 0  Rows_examined: 38819\nSET timestamp=1450281983;\nselect t1.table_name,t2.column_name,\"null\" as extra from information_schema.tables t1 left join information_schema.columns t2 on t1.table_name=t2.table_name and (t2.column_name =\"create_time\" or t2.column_name =\"test_create_time\") where t1.table_schema=\"schemareview_1450281983\" and t2.column_name is null union select t1.table_name,t2.column_name,\"null\" as extra from information_schema.tables t1 left join information_schema.columns t2 on t1.table_name=t2.table_name and (t2.column_name =\"update_time\" or t2.column_name =\"test_update_time\") where t1.table_schema=\"schemareview_1450281983\" and t2.column_name is null;",
      "@version" => "1",
    "@timestamp" => "2015-12-23T05:53:18.539Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/slow3306.log2",
          "type" => "mysql_slowlog",
          "tags" => [
        [0] "multiline"
    ]
}
```