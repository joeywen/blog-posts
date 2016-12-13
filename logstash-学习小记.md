\# logstash 学习小记



标签（空格分隔）： 日志收集 



---

\#\# Introduce

Logstash is a tool for managing events and logs. You can use it to collect logs, parse them, and store them for

later use \(like, for searching\). -- http://logstash.net



自从2013年logstash被ES公司收购之后，ELK stask正式称为官方用语，很多公司都开始ELK实践，我们也不例外，借用\[新浪是如何分析处理32亿条实时日志的？\]\(http://dockone.io/article/505\)的一张图

!\[此处输入图片的描述\]\[1\]

这是一个再常见不过的架构了：

（1）Kafka：接收用户日志的消息队列。

（2）Logstash：做日志解析，统一成JSON输出给Elasticsearch。

（3）Elasticsearch：实时日志分析服务的核心技术，一个schemaless，实时的数据存储服务，通过index组织数据，兼具强大的搜索和统计功能。

（4）Kibana：基于Elasticsearch的数据可视化组件，超强的数据可视化能力是众多公司选择ELK stack的重要原因。



但是众多log 收集framwork，像flume，scribe，fluent，为什么选用logstash呢？



原因很简单：



 1. 部署启动很容易，只需要有jdk就OK了

 2. 配置简单，无需编码

 3. 支持收集log路径的正则表达式，不像flume那样必须写死要收集的文件名，logstash不是，像这样

    &gt; path =&gt; \["/var/log/\*.log\*"\]





 有个\[Flume VS Fluentd VS Logstash\]\(https://everystack.io/\#!/compare/components/logstash\_vs\_fluentd\_vs\_flume\)可以看看

  \[1\]: http://dockerone.com/uploads/article/20150715/26d7dde91a3f96c858a5f4157ade2bf2.png

  

\#\# Logstash Examples

logstash事件处理流程氛围三个stages：input ，filter，output。input支持很多，如file，redis，kafka等等，filter主要是对input的log进行自己想要的处理，output则是输出到你要存储log的第三方framework，如kafka，redis，elasticsearch，db什么的，具体的查看官网。

废话不多说，开始例子：

1. 最最简单的例子

input和output都是标准输入输出

\`\`\` shell

\[joeywen@192 logstash\]$ bin/logstash -e 'input { stdin { } } output { stdout {}}'



Logstash startup completed

&gt;hello world  \#\# 输入的内容

&gt;2015-08-02T05:26:55.564Z joeywens-MacBook-Pro.local hello world   \#\# logstash收集的内容

\`\`\`





2. 编写config文件

\`\`\`config

input {

  file {

    path =&gt; \["/var/log/\*.log"\]

    type =&gt; "syslog"

    codec =&gt; multiline {

      pattern =&gt; "\(^\d+\serror\)\|\(^.+Exception:.+\)\|\(^\s+at .+\)\|\(^\s+... \d+ more\)\|\(^\s\*Causedby:.+\)"

      what =&gt; "previous"

    }

  }

}





output {

  stdout {  codec =&gt; rubydebug }

  \#  elasticsearch {

  \#      host =&gt; 'localhost'

  \#      protocol =&gt; 'transport'

  \#      cluster =&gt; 'elasticsearch'

 \#       index =&gt; 'logstash-joeymac-%{+YYYY.MM.dd}'

 \#   }

}

\`\`\`

输入是file形式，收集系统日志，如果有异常发生，通常异常会多行，这里用codec ＝&gt; multiline  来对出现异常的多行转换为一行输入



输出就是ES，或者你也可以把stdout作为调试打开看看，输出的是什么内容。运行命令如下以及输出



\`\`\`shell

\[joeywen@192 logstash\]$ bin/logstash -f sys.conf

Logstash startup completed



{

    "@timestamp" =&gt; "2015-08-02T05:36:08.972Z",

       "message" =&gt; "Aug  2 13:36:08 joeywens-MacBook-Pro.local GoogleSoftwareUpdateAgent\[1976\]: 2015-08-02 13:34:51.764 GoogleSoftwareUpdateAgent\[1976/0xb029b000\] \[lvl=2\] -\[KSUpdateEngine\(PrivateMethods\) updateFinish\] KSUpdateEngine update processing complete.",

      "@version" =&gt; "1",

          "host" =&gt; "joeywens-MacBook-Pro.local",

          "path" =&gt; "/var/log/system.log",

          "type" =&gt; "syslog"

}

{

    "@timestamp" =&gt; "2015-08-02T05:36:08.973Z",

       "message" =&gt; "Aug  2 13:36:08 joeywens-MacBook-Pro.local GoogleSoftwareUpdateAgent\[1976\]: 2015-08-02 13:36:08.105 GoogleSoftwareUpdateAgent\[1976/0xb029b000\] \[lvl=3\] -\[KSAgentUploader fetcher:failedWithError:\] Failed to upload stats to &lt;NSMutableURLRequest https://tools.google.com/service/update2&gt; with error Error Domain=NSURLErrorDomain Code=-1001 \"The request timed out.\" UserInfo=0x3605f0 {NSErrorFailingURLStringKey=https://tools.google.com/service/update2, \_kCFStreamErrorCodeKey=60, NSErrorFailingURLKey=https://tools.google.com/service/update2, NSLocalizedDescription=The request timed out., \_kCFStreamErrorDomainKey=1, NSUnderlyingError=0x35fd30 \"The request timed out.\"}",

      "@version" =&gt; "1",

          "host" =&gt; "joeywens-MacBook-Pro.local",

          "path" =&gt; "/var/log/system.log",

          "type" =&gt; "syslog"

}

{

    "@timestamp" =&gt; "2015-08-02T05:36:08.973Z",

       "message" =&gt; "Aug  2 13:36:08 joeywens-MacBook-Pro.local GoogleSoftwareUpdateAgent\[1976\]: 2015-08-02 13:36:08.272 GoogleSoftwareUpdateAgent\[1976/0xb029b000\] \[lvl=3\] -\[KSAgentApp uploadStats:\] Failed to upload stats &lt;KSStatsCollection:0x4323e0 path=\"/Users/joeywen/Library/Google/GoogleSoftwareUpdate/Stats/Keystone.stats\", count=6, stats={",

      "@version" =&gt; "1",

          "host" =&gt; "joeywens-MacBook-Pro.local",

          "path" =&gt; "/var/log/system.log",

          "type" =&gt; "syslog"

}

\`\`\`

如果想添加或删除字段，该怎么办？filter就该登场了



3. filter

filter的功能十分强大，可以对input的内容做任何更改，input的内容会转换为一个叫event的map，里面存放着key／value对，正如你所看到的输出一样，@timestamp,type,@version, host,message等等，都是event里面的key，在filter里面你可以启动ruby 编程plugin对其做任何更改

如：

\`\`\`

input {

  file {

    path =&gt; \["/var/log/\*.log"\]

    type =&gt; "syslog"

    codec =&gt; multiline {

      pattern =&gt; "\(^\d+\serror\)\|\(^.+Exception:.+\)\|\(^\s+at .+\)\|\(^\s+... \d+ more\)\|\(^\s\*Causedby:.+\)"

      what =&gt; "previous"

    }

  }

}



filter {



    if \[type\] =~ /^syslog/ {

        ruby {

            code =&gt; "file\_name = event\['path'\].split\('/'\)\[-1\]

            event\['file\_name'\] = file\_name"

        }

    }

}



output {

  stdout {  codec =&gt; rubydebug }

 }

\`\`\`

如上我对type已syslog开头的event做更改，调用ruby编程

看看输出

\`\`\`

\[joeywen@192 logstash\]$ bin/logstash -f sys.conf

Logstash startup completed

{

    "@timestamp" =&gt; "2015-08-02T05:46:52.771Z",

       "message" =&gt; "Aug  2 13:46:40 joeywens-MacBook-Pro.local Dock\[234\]: CGSConnectionByID: 0 is not a valid connection ID.",

      "@version" =&gt; "1",

          "host" =&gt; "joeywens-MacBook-Pro.local",

          "path" =&gt; "/var/log/system.log",

          "type" =&gt; "syslog",

     "file\_name" =&gt; "system.log"

}

\`\`\`

可以看到多了个file\_name的字段，

如果相对message做解析的话，需要调用grok plugin来做，grok是很强大插件,例如

\`\`\`

input {

  file {

    path =&gt; "/var/log/http.log"

  }

}

filter {

  grok {

    patterns\_dir =&gt; \["/opt/logstash/patterns", "/opt/logstash/extra\_patterns"\]

    match =&gt; { "message" =&gt; "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}" }

  }

}

\`\`\`

对于message字段调用正则匹配，语法是%{SYNTAX:SEMANTIC}

第一个SYNTAX是正则表达式名称，第二个是对于匹配成功的字段取名字，这些SYNTAX存在指定的pattern\_dir目录下的文件，格式是：

&gt;NAME PATTERN

&gt;如 NUMBER \d+



也可以使用mutate来最event的key和value做更改，包括remove，add，update,rename 等等，具体的都可以看看\(logstash文档\)\[https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html\]



这里给个具体的例子吧

配置：

\`\`\`

input {

  file {

    path =&gt; \["/var/log/\*.log"\]

    type =&gt; "syslog"

    codec =&gt; multiline {

      pattern =&gt; "\(^\d+\serror\)\|\(^.+Exception:.+\)\|\(^\s+at .+\)\|\(^\s+... \d+ more\)\|\(^\s\*Causedby:.+\)"

      what =&gt; "previous"

    }

  }

}



filter {



    if \[type\] =~ /^syslog/ {

        ruby {

            code =&gt; "file\_name = event\['path'\].split\('/'\)\[-1\]

            event\['file\_name'\] = file\_name"

        }

        grok {

            patterns\_dir =&gt; \["./patterns/\*"\]

            match =&gt; {"message" =&gt; "%{MAC\_BOOK:joeymac}"}

        }



        mutate {

            rename =&gt; {"file\_name" =&gt; "fileName"}

            add\_field =&gt; {"foo\_%{joeymac}" =&gt; "Hello world, from %{host}"}

        }

    }

}



output {

  stdout {  codec =&gt; rubydebug }

 }

\`\`\`

输出

\`\`\`

\[joeywen@192 logstash\]$ bin/logstash -f sys.conf

Logstash startup completed

{

                  "@timestamp" =&gt; "2015-08-02T06:10:13.161Z",

                     "message" =&gt; "Aug  2 14:10:12 joeywens-MacBook-Pro com.apple.xpc.launchd\[1\] \(com.apple.quicklook\[2206\]\): Endpoint has been activated through legacy launch\(3\) APIs. Please switch to XPC or bootstrap\_check\_in\(\): com.apple.quicklook",

                    "@version" =&gt; "1",

                        "host" =&gt; "joeywens-MacBook-Pro.local",

                        "path" =&gt; "/var/log/system.log",

                        "type" =&gt; "syslog",

                     "joeymac" =&gt; "joeywens-MacBook-Pro",

                    "fileName" =&gt; "system.log",

    "foo\_joeywens-MacBook-Pro" =&gt; "Hello world, from joeywens-MacBook-Pro.local"

}

\`\`\`









