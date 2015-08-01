# 用Bash Script编写Hadoop MapReduce Streaming

标签（空格分隔）： hadoop mapreduce bash

---

MapReduce对外提供一个多语言编写MR的功能，就是Hadoop Streaming。我们可以通过自己喜欢的语言来编写Mapper和Reducer函数，运行MapReduce job。

根据Hadoop Streaming的定义，只要我们能够从标准输入（standard input）读入数据，然后从标准输出（standard output）读出数据就OK了。但是有一点需要记住，就是如果你要使用自己喜欢的的语言，如Python，就必须要事先在集群上安装该语言对应的版本和对应的lib等等。这里给出Shell Script的示例

输入事文本文件，功能是从特定的字符开始统计单词的平均长度。你可以在程序里实现做些检查来忽略一些字符，也可以少用Pipes和一些command来提升性能等。

 1. Mapper Script ： word_lenght.sh
 

    

 ```bash
 #!/bin/bash
 #This mapper script will read one line at a time and  then break it into words
 #For each word starting LETTER and LENGTH of the     word are emitted
 while read line
 do
    for word in $line do
    if [ -n $word ] then
	    wcount=`echo $word | wc -m`;
	    wlength=`expr $wcount - 1`;
	    letter=`echo $word | head -c1`;
	    echo -e "$letter\t$wlength";
    fi
 done
 done
 #The output of the mapper would be “starting letter  of each word” and “its length”, separated by a tab space.
 ```
2. Reducer Script: avg_word_length.sh

    
```bash
#!/bin/bash
#This reducer script will take-in output from the mapper and emit starting letter of each word and average length
#Remember that the framework will sort the output from the mappers based on the Key
#Note that the input to a reducer will be of a form(Key,Value)and not (Key,
#This is unlike the input i.e.; usually passed to a reducer written in Java.
lastkey="";
count=0;
total=0;
iteration=1
while read line
 do
  newkey=`echo $line | awk '{print $1}'`
  value=`echo $line | awk '{print $2}'`
   if [ "$iteration" == "1" ] then
	lastkey=$newkey;c
	iteration=`expr $iteration + 1`;
   fi
   if [[ "$lastkey" != "$newkey" ]] then
	average=`echo "scale=5;$total / $count" | bc`;
	echo -e "$lastkey\t$average"
	count=0;
	lastkey=$newkey;
	total=0;
	average=0;
   fi
   total=`expr $total + $value`;
   lastkey=$newkey;
   count=`expr $count + 1`;
done
#The output would be Key,Value pairs(letter,average length of the words starting with this letter)
```

3. run command

    
``` shell
hadoop jar /usr/lib/hadoop-0.20-mapreduce/contrib/streaming/hadoop-streaming*.jar 
-input /input -output /avgwl 
-mapper mapper.sh 
-reducer reducer.sh 
-file /home/user/mr_streaming_bash/mapper.sh 
-file /home/user/mr_streaming_bash/reducer.sh 
```

还可以用其他语言鞋mapreduce来对比一下性能

翻译：[Hadoop MapReduce Streaming Using Bash Script ](http://princetonits.com/technology/hadoop-mapreduce-streaming-using-bash-script/)

>Google 大牛的Github Project
> [mapreduce in bash](https://github.com/erikfrey/bashreduce)