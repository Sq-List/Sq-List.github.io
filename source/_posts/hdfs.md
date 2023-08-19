---
title: hdfs
date: 2022-07-11 10:23:09
tags: 
    - hadoop
    - hdfs
categories:
	- hadoop
---

# 启蒙

## 基础知识

* 内存寻址的速度比IO寻址快10万倍
* IO速度大概在500MB每秒~1-2GB每秒（根据磁盘类型区分）
* **分而治之**
* 并行计算
* 计算要往数据移动
* 数据本地化读取
* 为什么hadoop项目中还要开发一个分布式文件系统？
  为了**更好的支持分布式计算**

# HDFS

## 存储模型

* 文件线性按字节切割成块（block），具有offset，id
* 文件与文件的block大小可以不一样
* 一个文件除最后一个block，其他block大小一致
* block的大小依据硬件的I/O特性调整
  * 1.X 64MB  
  * 2.X 128MB
  * 256MB
* block被分散存放在集群的节点中，具有location
* block具有副本（replication），没有主从概念，副本不能出现在同一个节点
* 副本是满足可靠性和性能的关键
* 文件上传可以指定block大小和副本数，上传后只能修改副本数
* 一次写入多次读取，**不支持修改**
* 支持追加数据

## 架构设计

* 主从（master/slaves）架构
* 由一个NameNode和一些DataNode组成
* 面向文件包含：文件数据（data）和文件元数据（metadata）
* NameNode（主）负责存储和管理文件元数据，并维护了一个层次性的文件目录树
* DataNode（从）负责存储文件数据（block块），并提供block的读写
* DataNode与NameNode维持心跳，并汇报自己持有的block信息
* Client与NameNode交互文件元数据和DataNode交互文件block数据

![HDFS架构](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/HDFS%E6%9E%B6%E6%9E%84-20220711-17:03:23.png)

![HDFS-block副本](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/HDFS-block%E5%89%AF%E6%9C%AC-20220711-17:06:18.png)

## 角色（进程）功能

### NameNode

* 完全基于内存存储文件元数据、目录结构、文件block的映射
* 需要持久化方案保证数据可靠性
* 提供**副本**放置策略

### DataNode

* 基于本地磁盘存储block（文件的形式）
* 保存block的校验和数据，保证block的可靠性
* 与NameNode保持心跳，汇报block列表状态

## 元数据持久化

* 任何对文件系统元数据产生修改的操作，NameNode都会使用一种称为EditLog的事务日志记录下来
* 使用FsImage存储内存所有的元数据状态
* 使用本地磁盘保存EditLog和FsImage

### 两种技术

#### 日志文件

* 完整性，数据丢失少
* 恢复数据慢，占空间

#### 镜像、快照

* 不能实时保存，需要间隔（小时、天）；容易丢失数据

* 恢复速度快过 日志文件

### 结合

* NameNode使用了FsImage+EditLog整合的方案：
  * 滚动将增量的EditLog更新到FsImage，以保证更近时点的FsImage和更小的EditLog体积

## 安全模式

* HDFS搭建时会格式化，格式化操作会产生一个空的FsImage。
  * ```hdfs namenode -format```

* 当NameNode启动时，它从硬盘中读取读取EditLog和FsImage。
* 将所有EditLog中的事务作用在内存中的FsImage上。
* 将这个新版本的FsImage从内存中保存到本地磁盘上，然后删除旧的EditLog（因为这个就的EditLog的事务都已经作用在FsImage上了）
* 
* NameNode启动后会进入一个称为安全模式的特殊状态。
* 处于安全模式的NameNode是不会进行数据块的复制的。
* NameNode会从所有DataNode接收心跳信号和块状态报告。
* 每当NameNode检测确认某个数据块的副本数量达到这个最小值，那么该数据块就会被认为是副本安全（safely replicated）的。
* 在一定百分比（这个参数可配置）的数据块被NameNode检测确认是安全之后（加上一个额外的30秒等待时间），NameNode将推出安全模式状态。
* 接下来它会确认还有那些数据块的副本没有达到指定数目，并将这些数据块复制到其他DataNode上。

## SecondaryNameNode（SNN）

* **在非HA模式（1.X没有HA模式）下**，SNN一般是独立的节点，周期完成对NN的EditLo向FsImage合并，减少EditLog大小，减少EditLog大小，减少NN启动时间
* 根据配置文件设置的时间间隔 fs.checkpoint.period 默认3600秒
* 根据配置文件设置EditLog大小 fs.checkpoint.size 规定EditLogs文件最大值默认64MB

![SNN](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/SNN-20220717-17:11:44.png)

## 副本放置策略

* 第一个副本：放置在上传文件的DataNode；如果是集群外提交，则随机挑选一台磁盘不太满，CPU不太忙的节点。
* 第二个副本：放置在与第一个副本不同的机架的节点上。
* 第三个副本：与第二个副本相同机架的节点。
* 更多副本：随机节点。

## 读写流程

### 写流程

![](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/HDFS%E5%86%99%E6%B5%81%E7%A8%8B-20220717-17:56:17.png)

* Client和NameNode连接创建文件元数据
* NameNode判定元数据是否有效
* NameNode处发副本放置策略，返回一个有序的DataNode列表
* Client和DataNode建立Pipeline连接
* Client将块切分成packet（64KB），并使用chunk（512B）+chucksum（4B）填充
* Client将packet放入发送队列dataqueue中，并向第一个DataNode发送
* 第一个DataNode收到packet后，本地保存并发送给第二个DataNode
* 第二个DataNode收到packet后，本地保存并发送给第三个DataNode
* 这一个过程中，上游节点同时发送下一个packet
* 生活中类比工厂的流水线；结论：流式其实也是变种的并行计算
* HDFS使用这种传输方式，副本数对于Client是透明的
* 当block传输完成，DataNode各自向NameNode汇报，同时Client继续传输下一个block
* 所以 Client的传输和block的汇报也是并行的
* Client只需要想建立连接的DataNode传输，不用管DataNode与DataNode间的传输（DataNode宕机之类 ），NameNode会保证集群中每个Block的副本数

### 读流程

![HDFS读流程](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/HDFS%E8%AF%BB%E6%B5%81%E7%A8%8B-20220724-10:37:17.png)

* 为了降低是整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本。
* 如果在读取程序的同一个机架上有一个副本。那么就读该副本。
* 如果一个HDFS集群跨越多个数据中心，那么客户端也将首先读本地数据中心的副本。
* 语义：下载一个文件
  * Client和NameNode交互文件元数据获取所有的fileBlockLocation
  * NameNode会按距离策略排序返回
  * Client尝试下载block并校验数据完整性
* 语义：下载一个文件其实是获取文件的所有block元数据，那么子集获取某些block也应该成立
  * <font color="#dd0000"> HDFS支持Client给出文件的offset自定义连接哪些block的DataName，自定义获取数据</font>
  * <font color="#dd0000"> 这个是支持计算层的分治，并行计算的核心</font>



### 其他

#### NameNode

* NameNode只有在重启集群后，可使用之前会将EditLog写入FsImage，其他在运行的情况下只能通过 SecondNameNode进行



## 目前存在问题

### 思路

* 主从集群：结构相对简单，主与从从协作
* 主：单点，数据一致好掌握
* 问题：
  * 单点故障，集群整体不可用
  * 压力过大，内存受限

### 解决方案

* 单点故障：
  * 高可用方案：HA
  * 多个NameNode、主备切换
  * 2.X提出HA 但是只支持一主一备；3.X能够一主多备，最多支持5备，建议3备
* 压力过大，内存受限：
  * 联邦机制：Federation（元数据分片）
  * 多个NameNode，管理不同的元数据



## HA

![HDFS-HA](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/HDFS-HA-20220831-17:34:56.png)

### Paxos算法

* 消息传递的一致性算法。
* 这个算法被认为是类似算法中最有效的。
* 该算法覆盖全部场景的一致性。
* 每种技术会根据子技术的特征选择简化算法实现。
* 传递：NN之间通过一个可靠的传输技术，最终数据能同步就可以

### 简化思路

* 分布式节点是否明确
* 节点权重是否明确
* 强一致性破坏可用性
* 过半通过 可以中和一致性和可用性
* 最简单的自我协调实现：主从
* 主的选举：明确节点数量和权重
* 主从的职能：
  * 主：增删改查
  * 从：查，增删改传递给主
  * 主与从：过半数同步数据

### 方案

* 多台NN主备模式，Active和Standby状态
  * Active对外提供服务
* 增加journalNode角色（>3台），负责同步NN的editlog
  * 最终一致性
* 增加zkfc角色（与NN同台），通过zookeeper集群协调NN的主从选举和切换
  * 事件回调机制
* DN同时向NN汇报block清单

### PS

* 开启了HA的话，SNN会被替换成standby NN，checkpoint过程会直接交给standby NN来负责。active NN会将edits文件同时写到本地与共享存储（比如QJM方案就是JournalNode集群）上去，standby NN从JournalNode集群拉取edits文件，和从active NN拉取的fsimage进行合并，并保持fsimage文件与active NameNode的同步。

## 部署流程

### 基础设施

#### ssh免密

1. 启动start-dfs.sh脚本的机器需要将公开的密钥分发给别的节点
2. 在HA模式下，每一个NN同机器会启动zkfc
   1. zkfc公用免密的方式控制自己和其他NN节点的NN状态

### 应用搭建

1. HA依赖ZK 搭建ZK集群
2. 修改hadoop的配置文件，并集群间同步

### 初始化启动

1. 先启动 JNN
   ```shell
   hadoop-daemon.sh start journalnode
   ```

2. 选择一个NN做格式化
   ```shell
   hdfs namenode format
   ```

3. 启动这个格式化的NN，以备另外几台同步
   ```shell
   hadoop-daemon.sh start namenode
   ```

4. 在另外几台NN中
   ```shell
   hdfs namenode -bootstrapStandby

5. 格式化ZK
   ```shell
   hdfs zkfc -formatZK
   ```

6. 启动hdfs
   ```shell
   start-dfs.sh
   ```

### PS

#### zookeeper三个端口的作用

* 2181：对cline端提供服务
* 3888：选举leader使用
* 2888：集群内机器通讯使用（Leader监听此端口）



## 权限

hdfs是一个文件系统

* 类unix、linux
* hdfs没有相关命令和接口去创建用户
  * 信任客户端 
    * 默认情况使用的 操作系统提供的用户
    * 扩展kerberos、LDAP （集成第三方用户认证系统）
  * 有超级用户的概念
    * linux系统重的超级用户：root
    * hdfs系统中超级用：namenode进程的启动用户
  * 权限概念
    * hdfs的权限是自己控制的 来自于hdfs的超级用户



## HDFS开发

```Java
@Test
public void blocks() throws Exception {

    Path file = new Path("/user/god/data.txt");
    FileStatus fss = fs.getFileStatus(file);
    BlockLocation[] blks = fs.getFileBlockLocations(fss, 0, fss.getLen());
    for (BlockLocation b : blks) {
        System.out.println(b);
    }
//        0,        1048576,        node04,node02  A
//        1048576,  540319,         node04,node03  B
    //计算向数据移动~！
    //其实用户和程序读取的是文件这个级别~！并不知道有块的概念~！
    FSDataInputStream in = fs.open(file);  //面向文件打开的输入流  无论怎么读都是从文件开始读起~！

//        blk01: he
//        blk02: llo msb 66231

    in.seek(1048576);
    //计算向数据移动后，期望的是分治，只读取自己关心（通过seek实现），同时，具备距离的概念（优先和本地的DN获取数据--框架的默认机制）
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
    System.out.println((char)in.readByte());
}
```



# MapReduce

## 计算模型

* Map：以一条记录为记录做映射

* Reduce：以一组为单位做计算
  * 什么叫做一组？按某种规律分组
  * 依赖一种数据格式 Key：Value
  * Key：Value 由map映射实现



* Map：
  * 映射、变换、过滤
  * 1进N出
* Reduce：
  * 分解、缩小、归纳
  * 一组近N出
* （KEY,VALUE）：
  * 键值对的键划分数据分组



切片

* 控制粒度（并行度）
* 一个切片对应一个map





![MapReduce执行流程](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/MapReduce%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B-20230709-17:00:59.png)

数据已一条记录为单位经过map方法映射成KV，相同key为一组，这一组数据调用一次reduce方法，在方法内迭代计算着一组数据。



* Bolck > split

  * 1:1

  * N:1

  * 1:N

* Split > map

  * 1:1

* Map > reduce

  * N:1
  * N:N
  * 1:1
  * 1:N

* Group(key) > partition

  * 1:1
  * N:1
  * N:N
  * *1:N*（只会进到其中一个partition）





### 细节

![MapReduce执行流程-细](https://cdn.jsdelivr.net/gh/sq-list/oss@master/uPic/MapReduce%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B-%E7%BB%86-20230709-17:04:50.png)

1. 切片会格式化出记录，以记录为单位调用map方法
2. map的输出映射成KV，KV会参与分区计算，拿着key算出P（分区号），得到KVP
   1. 内存缓存区大小默认128MB
3. 内存缓存区溢写时，做一个2次排序：分区有序，且分区内key有序，
4. mapTask的输出是一个文件，存在本地的文件系统中
   1. 如果文件不按分区排序，就会导致IO很高，每个reduce都会打开文件，重头遍历到尾
   2. 按照缓存区大小落地磁盘，就会得到一批内部有序外部无序的文件，再做一个归并排序即可得到一个内部有序的文件，每个reduce即可按序取一次即可
5. reduce的归并排序其实可以和reduce方法的计算同事发生，尽量减少IO
   1. 因为有迭代器模式的支持



迭代器模式是批量计算中非常优美的实现形式。













