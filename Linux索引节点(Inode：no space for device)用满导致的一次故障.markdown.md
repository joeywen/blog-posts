###问题描述
在storm测试环境集群上上nimbus和supervisor自动挂调，重启时显示no space for device，也不能创建，添加文件及目录，df －h查看

```shell

ilesystem            Size  Used Avail Use% Mounted on
/dev/vda1              40G  2.9G   35G   8% /
tmpfs                 3.9G     0  3.9G   0% /dev/shm
/dev/vdc1             100G  3.1G   92G   4% /home
/dev/vdd1              50G  180M   48G   1% /home/xxx/hard_disk/0
/dev/vde1              50G  180M   48G   1% /home/xxx/hard_disk/1
/dev/vdf1              50G  180M   48G   1% /home/xxx/hard_disk/2
/dev/vdg1              50G  180M   48G   1% /home/xxx/hard_disk/3
/dev/vdh1              50G  180M   48G   1% /home/xxx/hard_disk/4
```
显示空间使用很少，空间足够， df -i 显示

```shell

Filesystem            Inodes   IUsed   IFree IUse% Mounted on
/dev/vda1            2621440   75783 2545657    3% /
tmpfs                1007672       1 1007671    1% /dev/shm
/dev/vdc1             102400   85159   85159  100% /home
/dev/vdd1              51200      13   51187    1% /home/xxx/hard_disk/0
/dev/vde1              51200      18   51182    1% /home/xxx/hard_disk/1
/dev/vdf1              51200      18   51182    1% /home/xxx/hard_disk/2
/dev/vdg1              51200      18   51182    1% /home/xxx/hard_disk/3
/dev/vdh1              51200      18   51182    1% /home/xxx/hard_disk/4
```
发现inode使用率达到100%，这才找到问题
####什么是inode
####一、inode
硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存512字节（相当于0.5KB）。操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性读取一个"块"（block），每个"块"（block）由八个连续的sector组成。这种由多个扇区组成的"块"，是文件存取的最小单位。"块"的大小，最常见的是4KB，文件数据都储存在"块"中，那么还必须找到一个地方储存文件的元信息，比如文件的创建者、文件的创建日期、文件的大小等等。这种储存文件元信息的区域就叫做inode，中文译名为"索引节点"。每一个文件都有对应的inode，里面包含了与该文件有关的一些信息。
####二、inode的内容
inode包含文件的元信息，具体有以下内容：
　　* 文件的字节数
　　* 文件拥有者的User ID
　　* 文件的Group ID
　　* 文件的读、写、执行权限
　　* 文件的时间戳，共有三个：ctime指inode上一次变动的时间，mtime指文件内容上一次变动的时间，atime指文件上一次打开的时间。
　　* 链接数，即有多少文件名指向这个inode
　　* 文件数据block的位置 
可以用stat命令查看某个文件的inode信息：

```shell
[root@stream-e5s3c home]# stat apps
  File: `apps'
  Size: 4096            Blocks: 8          IO Block: 4096   directory
Device: fd02h/64770d    Inode: 4123        Links: 3
Access: (0700/drwx------)  Uid: (  456/    apps)   Gid: (  456/    apps)
Access: 2015-03-14 04:02:07.069500045 +0800
Modify: 2015-03-13 15:35:57.886985112 +0800
Change: 2015-03-13 15:36:28.209482045 +0800
```
####三、inode的大小
inode也会消耗硬盘空间，所以硬盘格式化的时候，操作系统自动将硬盘分成两个区域。一个是数据区，存放文件数据；另一个是inode区（inode table），存放inode所包含的信息。
每个inode节点的大小，一般是128字节或256字节。inode节点的总数，在格式化时就给定，一般是每1KB或每2KB就设置一个inode。假定在一块1GB的硬盘中，每个inode节点的大小为128字节，每1KB就设置一个inode，那么inode table的大小就会达到128MB，占整块硬盘的12.8%。
使用df -i可以查看每个硬盘分区的inode总数和已经使用的数量

###解决办法
进入到home路径下，用 ls  -i 查看每个路径所找用的信息：

```
   71 xxxxxxxx                     83969 xxxxxxxx      79877 xxxxxxxx  
   70 curator-recipes-2.5.0.jar       11 lost+found    86017 xxxxxxxx  
   68 curator.tar.gz               90113 mapred        88065 xxxxxxxx  
94209 xxxxxxxx                     79873 mobilereco    98305 xxxxxxxx  
   69 fds.jar                         55 ro_test       81921 vpp
69633 hdfs                         73729 spark         20481 yarn
92161 xxxxxxxx                     96257 xxxxxxxx  
  129 xxxxxxxx                      8193 storm
```
会在每个目录前面显示该路面所用的inode数目

如何释放inode信息，情查看下面信息

> It's quite easy for a disk to have a large number of inodes used even if the disk is not very full.
An inode is allocated to a file so, if you have gazillions of files, all 1 byte each, you'll run out of inodes long before you run out of disk.
It's also possible that deleting files will not reduce the inode count if the files have multiple hard links. As I said, inodes belong to the file, not the directory entry. If a file has two directory entries linked to it, deleting one will not free the inode.
Additionally, you can delete a directory entry but, if a running process still has the file open, the inode won't be freed.
My initial advice would be to delete all the files you can, then reboot the box to ensure no processes are left holding the files open.
If you do that and you still have a problem, let us know.
By the way, if you're looking for the directories that contain lots of files, this script may help:

```shell
#!/bin/bash
# count_em - count files in all subdirectories under current directory.
echo 'echo $(ls -a "$1" | wc -l) $1' >/tmp/count_em_$$
chmod 700 /tmp/count_em_$$
find . -mount -type d -print0 | xargs -0 -n1 /tmp/count_em_$$ | sort -n
rm -f /tmp/count_em_$$
```
or

```shell
sudo find . -xdev -type f | cut -d "/" -f 2 | sort | uniq -c | sort -n
```
list the files and remove these files to free inode.

参考文献
1. [Howto Free Inode Usage](http://stackoverflow.com/questions/653096/howto-free-inode-usage)
2. [Linux索引节点(Inode)用满导致的一次故障](http://loosky.net/2457.html)
3. [Linux的inode的理解](http://www.linuxidc.com/Linux/2014-09/106457p2.htm)