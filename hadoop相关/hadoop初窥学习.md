## Hadoop核心组件 ver 1.2x


### Hive

 - 功能：用于将SQL语句转化为可执行的Hadoop任务，降低了使用门槛

 ### Hbase
 

 - 类型：存储结构化数据的**分布式数据库** 
 - 区别：放弃传统关系型数据库事务的特性，追求更高的扩展。并且提供对数据的随机读写和实时的访问，实现对表数据的读写功能 
 
 ### Zookeeper
 

 - 功能：提供分布式一致性
### HDFS

 - 概念：Hadoop的文件系统，将所有文件以块的形式存储，默认块大小是64MB（ver 2.0+是128MB 可在hdfs-site.xml中修改dfs-block-size的属性）
 - 特点：（1）适用于大文件 （2）流式读写，即一次写入多次读 （3）备份冗余，硬件容错
	（4）高冗余，高容错
 - 主要节点：**NameNode和DataNode**
 - NameNode：管理节点，用于存放文件元数据。包括文件->数据块的映射表;数据块->数据节点的映射表.
 - DataNode:工作节点，存放真正的数据块
 - 保证可用的策略：**备份冗余、心跳检测、二级NameNode**
 - 备份冗余：每份数据块都有两份备份，即一共三个数据块，防止文件的丢失。同时三份数据块的存放策略是在同一份机器上会有两个数据块，在另外一个机器上存放另外一个数据块。
 - 心跳检测：datanode定时向namenode发送心跳包，保证机器正常运转
 - 二级NameNode：冷备份NameNode，保证元数据映射文件和修改日志
###MapReduce

 - 概念：Hadoop的计算框架，大多数的计算任务可以被抽象为：映射（分解）Map和规约Reduce过程。类似于JDK自带的Fork/Join框架
 - **容错机制**：（1）**尝试重试**，最多重试4次，超过则放弃 （2）**推测执行**，如果出现一个节点的计算远远比其他节点慢，那么新建一个节点执行同样的任务。谁先执行完，则放弃另外一个节点。这样防止不会因为单个节点故障而拖累整个任务的进度。
 - 重要名词：Job&Task，JobTracker,TaskTracker **注：这些在2.0+被移除，而交付由Yarn平台的ResourceManager统一管理，同时2.0在支持MapReduce之外还提供对Spark/Storm的支持**
 - Job&Task：Job是一个计算工作，一个Job可以被拆分为多个Task，任务。一般保证Job和Task与本机的dataNode对应，减少开销
 - JobTacker：（1）作业调度，类似于操作系统的调度算法，如FIFO,LRU。（2）分配并监控任务，用于将Job拆分为多个Task，同时观察各个任务的进度，防止有单个任务过慢（如出现单点故障）拖累整个计算进度。（3）监控TaskTacker
 - TaskTacker：（1）执行任务 （2）向JobTracker汇报任务执行进度
 - 


----------
### Hadoop Windows下安装踩过的坑
因为本身Hadoop在Linux环境下使用，移植到windows下面安装部署难免有多多少少的坑，将笔者遇到的坑写在这里供大家分享并提供解决的方案。

#### 配置文件相关

 - **core-site.xml:**这个笔者只配置了一个参数，需要注意的是网上搜到的大多文章默认给的配置是localhost:9000，如果直接搬运过去，可能在后面输入hdfs namenode -format的时候会抛出call from xxxx/ to localhost:9000 refuse 的异常，因为这里的localhost实际上应该替换成自己电脑的hostname，在cmd中输入hostname即可查看。


  
```
<configuration>
    <property>

          <name>fs.defaultFS</name>

        <value>hdfs://dell-pc:9000</value> //就是这里
    
    </property>

</configuration>
		
```


 - **hadoop-env.cmd**：这里会让大家进行一个JAVA_HOME的设定，笔者开始也是觉得只是一个简单的步骤，但是在进行hdfs namenode -format操作的时候发生报错。抛出了"JAVA_HOME set incorrectly"的错误。但是在cmd输入java/javac都是正确的，证明本地的JAVA_HOME并没有错。
**原因**：Linux环境下不分区，所以在它并不知道Windows环境下我们是分区的，所以Hadoop只能“看见”和它处在同一个盘符下的文件。
**解决方案**：如果你的JAVA_HOME和笔者一样，都是和Hadoop的安装目录不在一个盘的，那么只需要将这一行配置文件里的JAVA_HOME改成和Hadoop安装路径同一个地方的JDK即可。



```
rem The java implementation to use.  Required.
@rem set JAVA_HOME=%JAVA_HOME%
set JAVA_HOME=D:\JDK\Java\jdk1.8.0_101
 -
```


 - **hadoop-env.sh**：如果上一个修改后还没有让它正确运转，那么你可能需要修改这个文件 的内容。再一次显式的告诉Hadoop你的JAVA_HOME路径
```

export JAVA_HOME=D:\JDK\Java\jdk1.8.0_101


```



 - **hdfs-site.xml**:这个文件主要是进行对namenode和datanode存放目录文件的配置。如果这里配置不小心，可能启动start-dfs.cmd之后很快就会shutdown并报错(start-all.cmd已经被标记为废弃)
 **原因**:Linux下的分隔符和Windows下不一样
**解决方式**：分隔符如下面代码块所示，**记得一定要在盘符面前再加上一个'/'**

```
<configuration>
    <!-- Site specific YARN configuration properties -->
    <property>

        <name>dfs.replication</name>

        <value>1</value>

    </property>

    <property>

        <name>dfs.namenode.name.dir</name>//注意下面一行

        <value>/D:/codeWareCollections/hadoop-2.9.1/data/namenode</value>//一定要注意这里！！！
        

    </property>

    <property>

        <name>dfs.datanode.data.dir</name>

        <value>/D:/codeWareCollections/hadoop-2.9.1/data/datanode</value>

    </property>
</configuration>
```

------


## Hadoop执行相关

### Hadoop执行过程

 1. Split(分片）：该阶段执行在输入文件之后，将文件按默认的块大小（1.0是64MB，2.0是128MB）分成多个块储存在dataNode中
 2. Map（映射）+Combine（组合）:

	

 - map需要编码实现，对数据进行映射处理。每一个分片对应一个mapTask
 -Combine本质上是对Mapper缓冲区文件的合并，即一种本地的规约。为了减少过程中网络混洗的压力。在默认状态下，可以认为Combine是本地单击的reduce，执行的是reduce的逻辑；当然这个可以通过job.setCombinerClass进行设定。


	
 3. Shuffle（交换）:通过交换，在前面步骤中相同key值的数据通过交换，从不同的mapper中进入相同的partition
 4. Partition+Reduce（规约）：

	

 - partition发生在reduce之前。在默认状态下，如果有key值，那么输出结果会按照升序进行排序。相同的key值会进入同一个partition中。
 - reduce：规约输出最终结果的过程，在大数据量情况下不宜设计过少。partition，reducer和最终的输出文件(part-r-000XX)数量是1:1:1关系 
###Hadoop的分布式缓存
 - 概念：在执行mapreduce过程中，可能Mapper之间需要通信。在数据量不大的情况下，可以通过分布式缓存，将需要通信的文件从HDFS加载到本地内存中。笔者的个人理解，这实际上是一种类似于threadlocal的方式。
 - 发生时间：发生在job执行任务之前，保证能够读取需要的内容。本机上每个dataNode子节点都会缓存一份相同的共享数据。分布式缓存一般适用于较小的配置文件等等情形，如果文件过大，可以将文件分批缓存，重复执行作业
 - 使用方法:版本不同使用的方法不一致，只在这里提一下，可以使用#+名字的方式为缓存的文件设置别名。

 


 

 

