分布式 ID 生成系统，顾名思义，是在分布式的架构环境中，生成全局唯一标识的系统。比如在常见的业务系统中，
分布式 ID 可用来标记客户号、订单号、文件号、优惠券号等，以保证这些数据的全局唯一性。正如前文所述，

分布式架构下，唯一序列号生成是我们在设计一个系统，尤其是数据库使用分库分表的时候常常会遇见的问题。
当分成若干个sharding表后，如何能够快速拿到一个唯一序列号，是经常遇到的问题。

特性需求

全局唯一
支持高并发
能够体现一定属性
高可靠，容错单点故障
高性能

生成ID的方法有很多，来适应不同的场景、需求以及性能要求。

常见方式有：

1、利用数据库递增，全数据库唯一。

优点：明显，可控。
缺点：单库单表，数据库压力大。

2、UUID， 生成的是length=32的16进制格式的字符串，如果回退为byte数组共16个byte元素，即UUID是一个128bit长的数字，一般用16进制表示。

优点：对数据库压力减轻了。
缺点：但是排序怎么办？

此外还有UUID的变种，增加一个时间拼接，但是会造成id非常长。

3、twitter在把存储系统从MySQL迁移到Cassandra的过程中由于Cassandra没有顺序ID生成机制，于是自己开发了一套全局唯一ID生成服务：Snowflake。

1  41位的时间序列（精确到毫秒，41位的长度可以使用69年）
2  10位的机器标识（10位的长度最多支持部署1024个节点） 
3  12位的计数顺序号（12位的计数顺序号支持每个节点每毫秒产生4096个ID序号） 最高位是符号位，始终为0。

优点：高性能，低延迟；独立的应用；按时间有序。
缺点：需要独立的开发和部署。

4、Redis生成ID

当使用数据库来生成ID性能不够要求的时候，我们可以尝试使用Redis来生成ID。这主要依赖于Redis是单线程的，所以也可以用生成全局唯一的ID。可以用Redis的原子操作INCR和INCRBY来实现。

可以使用Redis集群来获取更高的吞吐量。假如一个集群中有5台Redis。可以初始化每台Redis的值分别是1,2,3,4,5，然后步长都是5。各个Redis生成的ID为：

A：1,6,11,16,21
B：2,7,12,17,22
C：3,8,13,18,23
D：4,9,14,19,24
E：5,10,15,20,25

比较适合使用Redis来生成每天从0开始的流水号。比如订单号=日期+当日自增长号。可以每天在Redis中生成一个Key，使用INCR进行累加。

优点：
不依赖于数据库，灵活方便，且性能优于数据库。
数字ID天然排序，对分页或者需要排序的结果很有帮助。
使用Redis集群也可以防止单点故障的问题。
 
缺点：
如果系统中没有Redis，还需要引入新的组件，增加系统复杂度。
需要编码和配置的工作量比较大，多环境运维很麻烦，
在开始时，程序实例负载到哪个redis实例一旦确定好，未来很难做修改。


5．还有其他一些方案，比如京东淘宝等电商的订单号生成。因为订单号和用户id在业务上的区别，订单号尽可能要多些冗余的业务信息，比如：

滴滴：时间+起点编号+车牌号
淘宝订单：时间戳+用户ID
其他电商：时间戳+下单渠道+用户ID，有的会加上订单第一个商品的ID。

而用户ID，则要求含义简单明了，包含注册渠道即可，尽量短。

MongoDB的ObjectId

前4 个字节是从标准纪元开始的时间戳，单位为秒。时间戳，与随后的5 个字节组合起来，提供了秒级别的唯一性。由于时间戳在前，这意味着ObjectId 大致会按照插入的顺序排列。这对于某些方面很有用，如将其作为索引提高效率。这4 个字节也隐含了文档创建的时间。绝大多数客户端类库都会公开一个方法从ObjectId 获取这个信息。

接下来的3 字节是所在主机的唯一标识符。通常是机器主机名的散列值。这样就可以确保不同主机生成不同的ObjectId，不产生冲突。 
为了确保在同一台机器上并发的多个进程产生的ObjectId 是唯一的，接下来的两字节来自产生ObjectId 的进程标识符（PID）。

前9 字节保证了同一秒钟不同机器不同进程产生的ObjectId 是唯一的。
后3 字节就是一个自动增加的计数器，确保相同进程同一秒产生的ObjectId 也是不一样的。同一秒钟最多允许每个进程拥有2563（16 777 216）个不同的ObjectId。

Twitter的snowflake算法
