### Hadoop分布式文件系统
当数据集的大小超过一台独立的物理计算机的存储能力时，就有必要对它进行分区并存储到若干台单独的计算机上。管理网络中跨多台计算机存储的文件系统称为分布式文件系统（distributed filesystem）。Hadoop自带一个称为HDFS的分布式文件系统，即Hadoop Distributed Filesystem。

#### 3.1 HDFS的设计
HDFS以流式数据访问模式来存储超大文件，运行于商用硬件集群上
- **超大文件**
这里指具有几百MB、几百GB甚至几百TB大小的文件，目前已经有存储PB级数据的Hadoop集群了
- **流式数据访问**
HDFS的构建思路是：一次写入、多次读取是最高效的访问模式。数据集通常由数据源生成或从数据源复制而来，接着长时间在此数据集上进行各种分析。读取整个数据集的时间延迟比读取第一条记录的时间延迟更重要
- **商用硬件**
Hadoop是设计运行在商用硬件的集群上的，对于庞大的集群来说，HDFS遇到节点故障被设计成能够继续运行且不让用户察觉到明显的中断
- **低时间延迟的数据访问**
要求低时间延迟数据访问的应用，例如几十毫秒范围，不适合在HDFS上运行。HDFS是为高数据吞吐量应用优化的，可能会以提高时间延迟为代价，对于低延迟的访问需求，HBase是更好的选择
- **大量的小文件**
namenode将文件系统的元数据存储在内存中，因此该文件系统所能存储的文件总数受限于namenode的内存容量
- **多用户写入，任意修改文件**
HDFS中文件写入只支持单个写入者，而且写操作总是以“只添加”方式在文件末尾写数据。它不支持多个写入者的操作，也不支持在文件的任意位置进行修改

#### 3.2 HDFS的概念
##### 3.2.1 数据块
- 每个磁盘都有默认的数据块大小，这是磁盘进行数据读/写的最小单位。构建于单个磁盘之上的文件系统通过磁盘块来管理该文件系统中的块，该文件系统块的大小可以是磁盘块的整数倍
- HDFS同样也有块（block）的概念，但是大得多，默认为128MB。与面向单一磁盘文件系统不同的是，HDFS中小于一个块大小的文件不会占据整个块的空间（例如，1MB的文件存储在一个128MB的块中时，文件只使用1MB的磁盘空间）
- 对分布式文件系统中的块进行抽象有几个好处。
**第一**，一个文件的大小可以大于网络中任意一个磁盘的容量。文件的所有块可以利用集群上的任意一个磁盘进行存储。
**第二**，使用抽象块而不是整个文件作为存储单元，简化了存储子系统的设计。将存储子系统的处理对象设置为块，可简化存储管理（块大小固定，计算单个磁盘能存储多少块很容易），也消除了对元数据的顾虑（元数据并不需要与块一同存储）。
**另外**，块还非常适合用于数据备份进而提供数据容错能力和提高可用性。如果发现一个块不可用，系统会从其他地方读取另一个复本。同时有些应用程序可能选择一些常用的文件块设置更高的复本数量进而分散集群中的读取负载
- `fsck`指令可显示块信息
`hdfs fsck / -files -blocks`将列出文件系统中各个文件由哪些块构成
#####3.2.2 namenode和datanode
HDFS集群有两类节点，即一个namenode（管理节点）和多个datanode（工作节点）
- namenode管理文件系统的命名空间，维护文件系统树及整棵树内所有的文件和目录。这些信息以两个文件形式永久保存在本地磁盘上：命名空间镜像文件和编辑日志文件
- datanode是文件系统的工作节点。它们根据需要存储并检索数据块，并且定期向namenode发送他们所存储的块的列表
- Hadoop对namenode的容错提供两种机制
    - 第一种是备份那些组成文件系统元数据持久状态的文件。一般的配置是，将持久状态写入本地磁盘的同时，写入一个远程挂载的网络文件系统（NFS）
    - 另一种可行的办法是运行一个辅助namenode，它的作用是定期合并编辑日志与命名空间镜像，以防止编辑日志过大，不作为namenode使用。辅助namenode一般在另一台单独的物理计算机上运行，保存合并后的命名空间镜像副本，并在namenode故障时启用。辅助namenode保存的状态总是滞后于主节点，为防止数据丢失，一般把存储在NFS上的namenode元数据复制到辅助namenode上并作为新namenode运行
##### 3.2.3 块缓存
对于频繁访问的文件，其对应的块可能被显式地缓存在datanode的内存中，以堆外块缓存（off-heap block cache）的形式存在，可以提高读操作的性能
##### 3.2.4 联邦HDFS
- namenode在内存中保存文件系统中每个文件和每个数据块的引用关系，这意味着内存将成为限制系统横向扩展的瓶颈。联邦HDFS允许系统添加namenode实现扩展，每个namenode管理文件系统命名空间中的一部分，例如一个管理/user，一个管理/share
- 联邦环境下，每个namenode维护一个命名空间卷（namespace volume），由命名空间的元数据和一个数据块池（block pool）组成，数据块池包含该命名空间下文件的所有数据块。命名空间卷相互独立，一个失效也不会影响其他维护的命名空间的可用性
- 要访问联邦HDFS集群，客户端需要使用客户端挂载数据表将文件路径映射到namenode。该功能可以通过`ViewFileSystem`和`viewfs://URI`进行配置和管理
##### 3.2.5 HDFS的高可用性
- 若namenode失效，Hadoop系统无法提供服务直到有新的namenode上线。这种情况下，要想从一个失效的namenode恢复，系统管理员得启动一个拥有文件系统元数据副本的新的namenode，并配置datanode和客户端以便使用这个新的namenode。对于一个大型并拥有大量文件和数据块的集群，namenode的冷启动需要30分钟，甚至更长时间
- Hadoop2针对上述问题增加了对HDFS高可用性的支持。配置了一对活动-备用（active-standby）namenode，当活动namenode失效，备用namenode就会接管它的任务并开始服务来自客户端的请求，不会有明显中断
    - namenode 之间需要通过高可用共享存储实现编辑日志的共享，当备用namenode接管工作之后通过通读共享编辑日志以实现状态同步，并继续读取由活动namenode写入的新条目
    - datanode 需要同时向两个namenode发送数据块处理报告
    - 客户端需要使用特定机制来处理namenode的失效问题，这一机制对用户是透明的
    - 辅助namenode的角色被备用namenode包含，备用namenode为活动的namenode命名空间设置周期性检查点
- 可从两种高可用性共享存储做出选择
    - NFS过滤器
    - 群体日志管理器（QJM，推荐）

#### 3.3 命令行接口
##### 文件系统的基本操作
- `hadoop fs -help`获取每个命令的详细帮助文件，命令`fs`的子命令与Linux系统中命令大体类似，子命令前面需要加`-`，如`hadoop fs -cat <src>`即展示指定文件内容
- `hadoop fs -copyFromLocal <localsrc> <dst>`从本地文件系统将一个文件复制到HDFS，`-copyFromLocal`与`-put`作用相同
- 其余命令查阅相关文档

#### 3.4 Hadoop文件系统
Hadoop有一个抽象的文件系统概念，HDFS只是其中的一个实现。Java抽象类`org.apache.hadoop.fs.FileSystem`定义了Hadoop中文件系统的客户端接口，该抽象类与Hadoop紧密相关的几个具体实现见下表
@import "./images/table3_1.png"
处理大数据集时，建议选择有数据本地优化的分布式文件系统，如HDFS

#### 3.5 Java接口
Hadoop是用Java写的，通过Java API可以调用大部分Hadoop文件系统的交互操作，本节主要探索FileSystem类：它是与Hadoop的某一文件系统进行交互的API（Hadoop2及后续版本新增FileContext文件接口，能更好处理多文件系统问题）
##### 从Hadoop URL读取数据
- 通过URLStreamHandler实例以标准输出方式显示Hadoop文件系统中的文件
    ```Java
    public class URLCat {
        static {
            URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory());
        }
        public static void main(String[] args) throws Exception {
            InputStream in = null;
            try {
                in = new URL(args[0]).openStream();
                IOUtils.copyBytes(in, System.out, 4096, false);
            } finally {
                IOUtils.closeStream(in);
            }
        }
    }
    ```
    使用java.net.URL对象打开数据流，通过FsUrlStreamHandlerFactory实例调用java.net.URL对象的setURLStreamHandlerFactory()方法使Java能够识别Hadoop的hdfs URL方案。
    缺点是该方法每个Java虚拟机只能调用一次，如果程序的其他组件已经声明一个URLStreamHandlerFactory实例，将无法用这种方法从Hadoop中读取数据
##### 通过FileSystem API读取数据
- 直接使用FileSystem以标准输出格式显示Hadoop文件系统中的文件
    ```Java
    public class FileSystemCat {
        public static void main(String[] args) throws Exception {
            String uri = args[0];
            Configuration conf = new Configuration();
            FileSystem fs = FileSystem.get(URI.create(uri), conf);
            InputStream in = null;
            try {
                in = fs.open(new Path(uri));
                IOUtils.copyBytes(in, System.out, 4096, false);
            } finally {
                IOUtils.closeStream(in);
            }
        }
    }
    ```
- Hadoop文件系统通过Hadoop Path对象来代表文件
- 获取FileSystem实例有下面几个静态工厂方法
`public static FileSystem get(Configuration conf) throws IOException`
`public static FileSystem get(URI uri, Configuration conf) throws IOException`
`public static FileSystem get(URI uri, Configuration conf, String user)
throws IOException`
Configuration
