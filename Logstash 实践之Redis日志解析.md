# Logstash 实践之Redis日志解析
## logstash config 脚本配置
``` ruby
input {
    file {
        path => ["/apps/svr/logstash/log/redis1.log"]
        start_position => "beginning"
        type => "redis_cluster"
    }
}

filter {
    multiline {
        what => next
        pattern => "^(?!(\d)+).*$"
    }
    grok {
        match => ["message", "(?<pid>.\d+?):(?<role>\w?)\s+(?<log_time>%{MONTHDAY}\s+%{MONTH}\s+%{HOUR}:%{MINUTE}:%{SECOND}?)\s+(?<log_level>.?)\s%{GREEDYDATA:message}"]
        overwrite => ["message"]
    }

    if [log_level] == "*" {
        mutate{ update => {"log_level" => "NOTICE"}}
    }

    if [log_level] == "#" {
        mutate{ update => {"log_level" => "WARNING"}}
    }

    if [log_level] == "-" {
        mutate{ update => {"log_level" => "VERBOSE"}}
    }

    if [log_level] == "." {
        mutate{ update => {"log_level" => "DEBUG"}}
    }
}
output {
    stdout {
        codec => rubydebug
    }

}
```
grok的正确性可以在该网站检验[Grok Test](http://grokconstructor.appspot.com/do/match)

Sample redis cluster log

```
230186:M 07 Jan 14:10:31.137 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:14:10.291 * FAIL message received from a36530f8df368550c186d9d8a2c5e39d3afe9b65 about 3d8bdc2e000031eb1f30f885d8a58fed4be270ed
230186:M 07 Jan 14:14:45.131 * Clear FAIL state for node 3d8bdc2e000031eb1f30f885d8a58fed4be270ed: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:15:27.525 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:15:27.525 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:15:58.911 * FAIL message received from a06d1ae1ca8dd7cfe504e63abec3edaf551ed708 about facf6a3b597646159ad54dd65ea8dbb47f43d570
230186:M 07 Jan 14:16:27.341 * FAIL message received from 8ac39fe250afc51a46ffeebbdd8e141c1a454b72 about 89bf2cb0a31daf2749c42acbfdc60652e5f42a4b
230186:M 07 Jan 14:16:29.250 * Clear FAIL state for node facf6a3b597646159ad54dd65ea8dbb47f43d570: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:16:57.682 * Clear FAIL state for node 89bf2cb0a31daf2749c42acbfdc60652e5f42a4b: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:18:11.971 * FAIL message received from d13e4e0ae910a367f8c221dba6758312c5ba1f6d about 77b4a19be1c7a6e02c4303431215d0f0b5ce2555
230186:M 07 Jan 14:18:39.389 * FAIL message received from cab133e37f569212ffb6ca92bbda103520caa907 about 7036b59d5949432e73ef7e026b9355b3b316e342
230186:M 07 Jan 14:18:43.400 * Clear FAIL state for node 77b4a19be1c7a6e02c4303431215d0f0b5ce2555: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:19:11.040 * Clear FAIL state for node 7036b59d5949432e73ef7e026b9355b3b316e342: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:19:34.438 * FAIL message received from 7036b59d5949432e73ef7e026b9355b3b316e342 about a06d1ae1ca8dd7cfe504e63abec3edaf551ed708
230186:M 07 Jan 14:20:01.984 * FAIL message received from db2fe945e25c1ca062ab4fc702d21d0ed823ee6d about a36530f8df368550c186d9d8a2c5e39d3afe9b65
230186:M 07 Jan 14:20:04.897 * Clear FAIL state for node a06d1ae1ca8dd7cfe504e63abec3edaf551ed708: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:20:33.030 * Clear FAIL state for node a36530f8df368550c186d9d8a2c5e39d3afe9b65: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:20:37.909 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:20:37.910 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:21:15.192 * FAIL message received from 8ac39fe250afc51a46ffeebbdd8e141c1a454b72 about 19e03002a74358cc71e1c95b2c0d6757fc2f9642
230186:M 07 Jan 14:22:09.279 * Clear FAIL state for node 19e03002a74358cc71e1c95b2c0d6757fc2f9642: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:25:25.381 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:26:29.850 * FAIL message received from 9f44d28db8692316dfae78935c7495f4b6ad74c7 about 9d4c4945f68130548480eef243a0b6b021addbf7
230186:M 07 Jan 14:27:00.704 * Clear FAIL state for node 9d4c4945f68130548480eef243a0b6b021addbf7: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:27:19.394 * FAIL message received from 42d11991231e06bdb4af300af2da97b3f97ce5cc about 694f2a76065554a1cb8eff899dc20be3587150ff
230186:M 07 Jan 14:27:52.868 * Clear FAIL state for node 694f2a76065554a1cb8eff899dc20be3587150ff: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:29:49.898 * FAIL message received from d9d8135bc34778c5efa7becb343c34ae39dbed0e about 3583cbbb16d17c23b833733aab3c580dca54cfbb
230186:M 07 Jan 14:30:20.431 * Clear FAIL state for node 3583cbbb16d17c23b833733aab3c580dca54cfbb: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:30:33.976 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:30:33.976 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:31:13.777 * Marking node 1d380cf0474b634a8584395d10cd931617f92906 as failing (quorum reached).
230186:M 07 Jan 14:31:45.018 * Clear FAIL state for node 1d380cf0474b634a8584395d10cd931617f92906: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 14:35:26.592 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 14:35:26.592 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 15:30:37.196 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 15:34:05.311 * Marking node 77b4a19be1c7a6e02c4303431215d0f0b5ce2555 as failing (quorum reached).
230186:M 07 Jan 15:34:42.468 * Clear FAIL state for node 77b4a19be1c7a6e02c4303431215d0f0b5ce2555: is reachable again and nobody is serving its slots after some time.
230186:M 07 Jan 15:35:29.251 # Bad message length or signature received from Cluster bus.
230186:M 07 Jan 15:35:29.252 # Bad message length or signature received from Cluster bus.
```

样例输出
``` xml
{
       "message" => "No cluster configuration found, I'm 40430deb258bee01b769490bd2cd21155f35f431",
      "@version" => "1",
    "@timestamp" => "2016-01-18T04:25:18.767Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/redis1.log",
          "type" => "redis_cluster",
           "pid" => "230186",
          "role" => "M",
      "log_time" => "07 Jan 09:08:39.824",
     "log_level" => "NOTICE"
}
{
       "message" => "Server started, Redis version 3.0.3",
      "@version" => "1",
    "@timestamp" => "2016-01-18T04:25:18.769Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/redis1.log",
          "type" => "redis_cluster",
          "tags" => [
        [0] "multiline"
    ],
           "pid" => "\n230186",
          "role" => "M",
      "log_time" => "07 Jan 09:08:39.825",
     "log_level" => "WARNING"
}
{
       "message" => "WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.",
      "@version" => "1",
    "@timestamp" => "2016-01-18T04:25:18.774Z",
          "host" => "joeywens-MacBook-Pro.local",
          "path" => "/apps/svr/logstash/log/redis1.log",
          "type" => "redis_cluster",
           "pid" => "230186",
          "role" => "M",
      "log_time" => "07 Jan 09:08:39.825",
     "log_level" => "WARNING"
}
```


