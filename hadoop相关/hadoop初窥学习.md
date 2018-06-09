## Hadoop核心组件 ver 1.2x


### Hive

 - 概念：交互式SQL，使得能在Hadoop上获得SQL查询的低延迟响应并保持大数据集规模的可扩展性。
 - 功能：用于将SQL语句转化为可执行的Hadoop任务，降低了使用门槛。


 ### Hbase
 

 - 类型：存储结构化数据的**分布式数据库** 
 - 概念：由HDFS作为底层的键值对存储模型，除对数据行的读写外，支持对数据块的读写。
 - **与关系型数据库区别**：放弃事务的特性，追求更高的扩展。并且提供对数据的随机读写和实时的访问，实现对表数据的读写功能。另外关系型数据的另外一个瓶颈是：<span id="bottonNeck">**寻址读写速度的提升远低于传输速度的提升**</span>；类似于MySQL的InnoDB引擎采用B+树作为数据结构，主要瓶颈是对硬盘读写的寻址速度。而大数据的流处理瓶颈则是数据的硬盘带宽（传输速度）
 
 ### Zookeeper
 

 - 功能：提供分布式一致性
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

 - **core-site.xml:**这个笔者只配置了一个参数，需要注意的是网上搜到的大多文章默认给的配置是localhost:9000，如果直接搬运过去，可能在后面输入hdfs namenode -format的时候会抛出call from xxxx/ to localhost:9000 refuse 的异常，因为这里的localhost实际上应该替换成自己电脑的hostname，在cmd中输入hostname即可查看。**这一步这样做实际上是为了搭建伪分布式**


  
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
        //这一步是强行设置，规避HDFS要求备份三份的要求，不然DataNode小于3会提示//报错
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

	

 - map需要编码实现，对数据进行映射处理。<span id="mapTask对应关系">每一个分片对应一个mapTask</span>
 -Combine本质上是对Mapper缓冲区文件的合并，即一种本地的规约。为了减少过程中网络混洗的压力。在默认状态下，可以认为Combine是本地单击的reduce，执行的是reduce的逻辑；当然这个可以通过job.setCombinerClass进行设定。


	
 3. Shuffle（交换）:通过交换，在前面步骤中相同key值的数据通过交换，从不同的mapper中进入相同的partition
 4. Partition(切分）+Reduce（规约）：partition决定k-v分配到哪一个reducer，因为我们的程序可能不止一个reducer。Hadoop默认使用Hash取模的方法。因此k-v的hashcode相同的会被分到同一个partition下，而partition=reducer=1：1.Reduce（*partition*）的数量不是根据输入数据决定的，而是独立指定的。
	

 - partition发生在reduce之前。在默认状态下，如果有key值，那么输出结果会按照升序进行排序。相同的key值会进入同一个partition中。
 - reduce：规约输出最终结果的过程，在大数据量情况下不宜设计过少。partition，reducer和最终的输出文件(part-r-000XX)数量是1:1:1关系 
###Hadoop的分布式缓存
 - 概念：在执行mapreduce过程中，可能Mapper之间需要通信。在数据量不大的情况下，可以通过分布式缓存，将需要通信的文件从HDFS加载到本地内存中。笔者的个人理解，这实际上是一种类似于threadlocal的方式。
 - 发生时间：发生在job执行任务之前，保证能够读取需要的内容。本机上每个dataNode子节点都会缓存一份相同的共享数据。分布式缓存一般适用于较小的配置文件等等情形，如果文件过大，可以将文件分批缓存，重复执行作业
 - 使用方法:版本不同使用的方法不一致，只在这里提一下，可以使用#+名字的方式为缓存的文件设置别名。
 ### 编码方法执行顺序
1. setUp():本方法是在mapReduce过程中最先执行的。通常用于mapTask的一些预处理过程，比如创建一个容器，如List
1. map()：最常重写的方法，接收输入分片的一次recordReader读取的k-v键值对，然后进行map处理
1. cleanup():本方法类似于finnaly，最后执行，用于收尾工作，如关闭流、释放资源，以及context.write写入数据
1. run(),在下面的代码块中可以看出，默认的run方法实际上是把1.2.3方法整合起来：先直接调用setup方法，然后每一次数据片调用map方法进行映射；最后调用cleanup
```
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
    ```
    public class LineRecordReader extends RecordReader<LongWritable, Text> {  
  private static final Log LOG = LogFactory.getLog(LineRecordReader.class);  
  
  private LongWritable key = null;  
  private Text value = null;  
    ```


 


 

 

