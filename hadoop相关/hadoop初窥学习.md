

- [Hadoop核心组件 ver 1.2x](#hadoop核心组件-ver-12x)
    - [Hive](#hive)
    - [Hbase](#hbase)
    - [Zookeeper](#zookeeper)
    - [HDFS](#hdfs)
    - [MapReduce](#mapreduce)
    - [Hadoop Windows下安装踩过的坑](#hadoop-windows下安装踩过的坑)
        - [配置文件相关](#配置文件相关)
        - [抛出的异常类型以及处理方法](#抛出的异常类型以及处理方法)
    - [Hadoop在Linux环境下安装(VMWare Ubuntu)踩的坑](#hadoop在linux环境下安装vmware-ubuntu踩的坑)
- [Hadoop执行相关](#hadoop执行相关)
    - [Hadoop执行过程](#hadoop执行过程)
    - [Hadoop的分布式缓存](#hadoop的分布式缓存)
    - [编码方法执行顺序](#编码方法执行顺序)
- [Hadoop核心类](#hadoop核心类)
    - [InputFormat](#inputformat)
    - [FileSystem](#filesystem)
    - [Configuration](#configuration)
## Hadoop核心组件 ver 1.2x


### Hive

 - 概念：交互式SQL，使得能在Hadoop上获得SQL查询的低延迟响应并保持大数据集规模的可扩展性。
 - 功能：用于将SQL语句转化为可执行的Hadoop任务，降低了使用门槛。


 ### Hbase
 

 - 类型：存储结构化数据的**分布式数据库** 
 - 概念：由HDFS作为底层的键值对存储模型，除对数据行的读写外，支持对数据块的读写。
 - **与关系型数据库区别**：放弃事务的特性，追求更高的扩展。并且提供对数据的随机读写和实时的访问，实现对表数据的读写功能。另外关系型数据的另外一个瓶颈是：<p id="bottonNeck">**寻址读写速度的提升远低于传输速度的提升**</p>；类似于MySQL的InnoDB引擎采用B+树作为数据结构，主要瓶颈是对硬盘读写的寻址速度。而大数据的流处理瓶颈则是数据的硬盘带宽（传输速度）
 
 ### Zookeeper
 - 功能：作为Hadoop的分布式协调服务，提供分布式一致性
 - 起因：分布式系统在通信时一种固有问题 _“部分失败”_，即消息的发送者并不能知道接受者是否正常、正确的收到了消息。
 - 特点:
    1. 简单，核心是一个精简的文件系统，提供排序和通知功能
    2. 主动通知，不是被动的。

- 文件系统：
    1. 特性：类似于数据结构中的多重表，由结点 _znode_ 组成，结点可以包含内容文件，也可以包含其他 _znode_ 
    2. znode:分为短暂和持久型,可以在zk.create中使用CreateMode的四种枚举类型指定。
        1. 短暂型：无论任何原因导致连接断开，短暂型的znode节点都会被Zookeeper删除
        2. 持久型：**仅当客户端主动删除(不一定是创建它的客户端)**，持久的znde才会被删除
- 编码相关：
    1.  创建znode实现： 
        1. Zookeeper构造方法有多个重载，核心参数是 _zookeeper主机地址，会话超时时间，Watcher对象实例_
        2. Watcher接口，其中有一个方法progress()，与前面类似，这是一个回调方法。接受来自于Zookeeper各种事件通知的回调。
        3. 由于 **构造方法是立刻返回的**，而这种开启服务的操作可能耗时较长，因此需要采用线程同步的方法，比如CountDownLatch,Thread.join()等方法使构造zookeeper实例的方法能够正确返回。
        4. 使用Zookeeper的实例方法create()创建所需的znode 
### HDFS

 - 概念：Hadoop的文件系统，将所有文件以块的形式存储，默认块大小是64MB（ver 2.0+是128MB 可在hdfs-site.xml中修改dfs-block-size的属性）
 - **为什么HDFS的块这么大**：*总的文件读时间=硬盘寻址+数据传输时间*；前面提到<a href="#bottonNeck">[现代计算机读写瓶颈]</a>。那么显然这是一种“寻址密集型”的情况，如果要提高效率，主要需要解决的是寻址的时间，则我们将块尽量的设置大一点，减少寻址时间即可。<a href="#mapTask对应关系">为什么不更大</a>，任务过少会导致大量的节点空闲，不能充分利用资源，影响整体的效率。
 - 特点：（1）适用于大文件 （2）流式读写，即一次写入多次读 （3）备份冗余，硬件容错
	（4）高冗余，高容错
 - 主要节点：**NameNode和DataNode**
 - NameNode：管理节点，用于存放文件元数据。包括文件->数据块的映射表;数据块->数据节点的映射表.
 - DataNode:工作节点，存放真正的数据块
 - 保证可用的策略：**备份冗余、心跳检测、二级NameNode**
 - 备份冗余：每份数据块都有两份备份，即一共三个数据块，防止文件的丢失。同时三份数据块的存放策略是在同一份机器上会有两个数据块，在另外一个机器上存放另外一个数据块。
 - 心跳检测：datanode定时向namenode发送心跳包，保证机器正常运转
 - 二级NameNode：热备份NameNode，保证元数据映射文件和修改日志。重启namenode的时间会非常长
### MapReduce


 - 概念：Hadoop的计算框架。提供一种框架，用于将分布式系统的读写操作抽象成**一个数据集**（由键值对组成）的计算。大多数的计算任务可以被抽象为：映射（分解）Map和规约Reduce过程。类似于JDK自带的Fork/Join框架
 - 执行流程：
```mermaid
graph LR
input-->map
map-->产生文件是否溢出缓存
产生文件是否溢出缓存-->是,溢写到磁盘
产生文件是否溢出缓存-->否,随reduce加入内存
是,溢写到磁盘-->超过3个文件combine组合
超过3个文件combine组合-->shuffle
否,随reduce加入内存-->reduce
shuffle-->reduce

```

 - **容错机制**：（1）**尝试重试**，最多重试4次，超过则放弃 （2）**推测执行**，如果出现一个节点的计算远远比其他节点慢，那么新建一个节点执行同样的任务。谁先执行完，则放弃另外一个节点。这样防止不会因为单个节点故障而拖累整个任务的进度。
 - 适用： 批量大文件查询、离线的使用场景（用户不需要等待，因为时间很长）
 - 重要名词：Job&Task，JobTracker,TaskTracker **注：这些在2.0+被移除，而交付由Yarn平台的ResourceManager统一管理，同时2.0在支持MapReduce之外还提供对Spark/Storm的支持**
 - Job&Task：Job是一个计算工作，一个Job可以被拆分为多个Task，任务。一般保证Job和Task与本机的dataNode对应，减少开销
 - JobTacker：（1）作业调度，类似于操作系统的调度算法，如FIFO,LRU。（2）分配并监控任务，用于将Job拆分为多个Task，同时观察各个任务的进度，防止有单个任务过慢（如出现单点故障）拖累整个计算进度。（3）监控TaskTacker
 - TaskTacker：（1）执行任务 （2）向JobTracker汇报任务执行进度
 - 


----------
### Hadoop Windows下安装踩过的坑
因为本身Hadoop在Linux环境下使用，移植到windows下面安装部署难免有多多少少的坑，将笔者遇到的坑写在这里供大家分享并提供解决的方案。

#### 配置文件相关

 - <p id="core-site">core-site.xml:</p>这个笔者只配置了一个参数，需要注意的是网上搜到的大多文章默认给的配置是localhost:9000，如果直接搬运过去，可能在后面输入hdfs namenode -format的时候会抛出call from xxxx/ to localhost:9000 refuse 的异常，因为这里的localhost实际上应该替换成自己电脑的hostname，在cmd中输入hostname即可查看。**这一步这样做实际上是为了搭建伪分布式**


  
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
        //这一步是强行设置，规避HDFS要求备份三份的要求，不然//DataNode小于3会提示报错
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
#### 抛出的异常类型以及处理方法
d 
1. java.net.ConnectException: Connection refused: no further information
    - 出处：
        1. 前面配置文件过程中[core-site](#core-site)中的主机名没有配置成hostname而是照抄的localhost等等
        1. 另外一种情况则是笔者昨天刚遇到的，我尝试做一个hdfs文件读取并使用FSDataInputStream特性seek()的demo。但是几行代码编写完之后，运行时却抛出了异常。
    - 解决方案：
        1. 第一种情况，按照前面提到的，将localhost改为自己的主机名(*hostname*)即可
        1. 第二种情况，首先检查*Windows 防火墙*是否关闭，或者是否开放了入站规则。其次，确认你已经成功启动hadoop，使用jps命令查看；然后，使用netstat -an|findstr "你core-site中配置的端口号" 命令查看是否端口已经建立或监听中。关键部分来了,笔者在原来的代码中，生成的URI是这样的,找了无数方法都连不上
     ```java
    String uri ="hdfs://dell-pc:9000/test.txt";

    ```
     后面通过netstat -an|findstr "9000",发现建立连接的两个端口是192.168.181.1:9000，因此将uri改成了如下，问题解决。

    ```java
    String uri =  "hdfs://192.168.181.1:8080/test.txt";

    ```
    ### Hadoop在Linux环境下安装(VMWare Ubuntu)踩的坑
    **写在前面**:笔者本身开发平常只在Windows下，但前段时间去某公司实习以及春招的经历，多多少少还是会用到Linux，所以干脆把这块骨头啃一啃。同时记录一下一些坑，方便后面查看。

    - Linux系统和Windows系统不太一样的一点，是这个系统上面下载东西比较困难，需要自己去更新软件源。这个操作比较简单，百度查一下就可以。需要注意的是，如果你和笔者一样，安装的版本不是*LTS（长期支持版本）*，那你需要查看一下这个版本是否依然被支持。像笔者前面下载的镜像是17.04，而这个版本不是LTS，在2018-1就已经停止支持。所以笔者换了好多次源都不能成功下载。
        **解决方案**：可以升级到例如17.10这种LTS版本
    - 注意权限问题，Linux系统对权限界定的比较死。像笔者遇到一个比较奇葩的问题是，在/etc/passwd里面将自己的权限提升到了0:0(root)之后导致火狐浏览器打不开了。
        **解决方案**：笔者将用户权限和root权限进行了分离使用..
    - Hadoop需要使用ssh登陆，这时候可能会出现情况 _permission denied_ 的情况，需要在config文件中修改，这个可以百度到。
**总结**：总体来说比Windows遇到的坑少很多，大多数问题通过搜索能比较快的解决。像配置文件中的内容坑和Windows差不太多。最后上一一下结果图哈哈。
@import "ubun-hadoop.png"
------


## Hadoop执行相关

### Hadoop执行过程

 1. Split(分片）：该阶段执行在输入文件之后，将文件按默认的块大小（1.0是64MB，2.0是128MB）分成多个块储存在dataNode中
 2. Map（映射）+Combine（组合）:

	

 - map需要编码实现，对数据进行映射处理。<b id="mapTask对应关系">每一个分片对应一个mapTask</b>
 -Combine本质上是对Mapper缓冲区文件的合并，即一种本地的规约。为了减少过程中网络混洗的压力。在默认状态下，可以认为Combine是本地单击的reduce，执行的是reduce的逻辑；当然这个可以通过job.setCombinerClass进行设定。


	
 3. Shuffle（交换）:通过交换，在前面步骤中相同key值的数据通过交换，从不同的mapper中进入相同的partition
 4. Partition(切分）+Reduce（规约）：partition决定k-v分配到哪一个reducer，因为我们的程序可能不止一个reducer。Hadoop默认使用Hash取模的方法。因此k-v的hashcode相同的会被分到同一个partition下，而partition=reducer=1：1.Reduce（*partition*）的数量不是根据输入数据决定的，而是独立指定的。
	

 - partition发生在reduce之前。在默认状态下，如果有key值，那么输出结果会按照升序进行排序。相同的key值会进入同一个partition中。
 - reduce：规约输出最终结果的过程，在大数据量情况下不宜设计过少。partition，reducer和最终的输出文件(part-r-000XX)数量是1:1:1关系 
### Hadoop的分布式缓存
 - 概念：在执行mapreduce过程中，可能Mapper之间需要通信。在数据量不大的情况下，可以通过分布式缓存，将需要通信的文件从HDFS加载到本地内存中。笔者的个人理解，这实际上是一种类似于threadlocal的方式。
 - 发生时间：发生在job执行任务之前，保证能够读取需要的内容。本机上每个dataNode子节点都会缓存一份相同的共享数据。分布式缓存一般适用于较小的配置文件等等情形，如果文件过大，可以将文件分批缓存，重复执行作业
 - 使用方法：直接可以调用*job.addCacheFile(URI uri)*/*addCachedArchives()*.可以使用#+名字的方式为缓存的文件设置别名。
### 编码方法执行顺序
1. setUp():本方法是在mapReduce过程中最先执行的。通常用于mapTask的一些预处理过程，比如创建一个容器，如List
1. map()：最常重写的方法，接收输入分片的一次recordReader读取的k-v键值对，然后进行map处理
2. cleanup():本方法类似于finnaly，最后执行，用于收尾工作，如关闭流、释放资源，以及context.write写入数据
3. run(),在下面的代码块中可以看出，默认的run方法实际上是把1.2.3方法整合起来：先直接调用setup方法，然后每一次数据片调用map方法进行映射；最后调用cleanup

```java{.line-numbers}
public void run(Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT>.Context context) throws IOException, InterruptedException {
        this.setup(context);

        try {
            while(context.nextKeyValue()) {
                this.map(context.getCurrentKey(), context.getCurrentValue(), context);
            }
        } finally {
            this.cleanup(context);
        }

    }
```

---

## Hadoop核心类
### InputFormat
- **特点**：这是一个抽象类，没有实现方法体。用于文件读取输入时的格式。默认是采用TextInputFormat，即以**文本行的形式读取数据，每一行为一个<key,value>键值对**。一般是<LongWritable,Text>，即行（偏移量）数和对应行中的文本。
- **核心参数及意义**：
    - InputSplits:这个参数值指的是文件被分成块的默认大小，一般是128MB;其List中包含的数量则是文件被分成块的数量。我们知道如果不是CPU密集型的mapReduce操作，那么一个块文件会对应MapTask。如果不希望一个文件被切分成多个切片，那么可以覆写isSplitalbe方法，直接放回false，那么一个文件将不会被切分，而是整个文件被一个mapTask处理   
    - RecordReader：如果说InputSplit是处理文件的分片，那么RecordReader则是处理一个InputSplit(分片)中的key-value键值对，使其能正确的被读出来。默认实现中，是以行偏移量为key,行内容为value。从下面代码段可以看出来，**默认的k-v类型是LongWriteable-Text**
    ```java
    public class LineRecordReader extends RecordReader<LongWritable, Text> {  
  private static final Log LOG = LogFactory.getLog(LineRecordReader.class);  
  
  private LongWritable key = null;  
  private Text value = null;  
    ```

### FileSystem
- 构造：通过静态工厂get()获得实例，有三个重载
- 重要方法：get()获得实例，open()打开文件I/O流，可以读写HDFS文件，这个I/O流支持前面提到的随意读写文件，可以输入一个文件的绝对位置来对文件进行访问；而不是像普通的java.io中的skip只能访问当前位置之后的相对位置。+
- 搭配:在上面通过open获得流之后，通过IOUtils封装的方法可以方便的进行读写操作
### Configuration
- 含义：封装了客户端或服务端设定的配置。默认构造实现是前面的[core-site.xml](#core-site)中的配置



 


 

 

