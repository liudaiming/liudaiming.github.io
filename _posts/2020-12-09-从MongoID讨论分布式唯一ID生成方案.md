---
layout: post
title: "从MongoID的生成讨论分布式唯一ID生成方案"
date: 2020-12-09
tags:
 - 分布式ID
 - MongoID
---

### 背景

>MongoDB，想必大家都使用过，在数据落盘后，查询该条数据时，会发现其会自动生成一条"_id"，如：
>
	db.test.insert({"name":"tom"})
	查询结果：
	{ "_id" : ObjectId("5fd049327fbb28868f4660a5"), "name" : "tom" }

>MongoID作为主键索引，即使是集群情况下，其在整个数据库中也是全局唯一的。


### 使用场景

那这种ID有什么用呢？或者说在哪些场景下会被使用呢？
当然需要满足下面两个条件：

1.数据量现在或未来比较大，处于增长状态。

2.需要用唯一ID来标识。

比如说订单，优惠券，消息，待检索或处理数据（地图poi数据，蜘蛛抓取的数据）等等。

### 特性

那这些场景ID需要什么特性呢：

>1.全局唯一。这是肯定的，两个不同订单生成了相同的ID，那怎么区分呢？

>2.趋势递增。生成的ID可能用作数据库索引，而一些数据库索引是B+树索引，如果趋势递增，则更能使用磁盘的块读取存储特性，提高写入性能。

>3.高性能，高可用。不能成为业务瓶颈。

>4.非连续。如果该ID对外使用，则很容易得知一天的数据量。如果是详情ID，也很容易被批量抓取。

>5.字节占用少。如果全由数字构成，则存储，排序更高效。如1111111比aaaaaa-111111在存储及排序方面更有优势
>

### MongoID构成


MongoID使用12个字节来表示,每个字节两位十六进制。如下图：

![](https://www.imflybird.cn/static/img/2020/mongoid.png)

而平时业务中需要先生成ID供业务方使用，那有哪些方案呢？

### 方案一：

使用UUID生成器（很多编程语言自带UUID生成）。生成格式为：xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
生成的格式是长字符串，如：123e4567-e89b-12d3-a456-426655440000，如果作为主键，存储及索引性能低。


### 方案二：

使用自增ID（用程序生成或借助mysql，redis）。但是会存在下面问题：

>1.不满足特性4的非连续，不过如果仅内部使用，倒无所谓。

>2.我们提到的是分布式，而这种方案如果要分布式提供服务，则运营成本很大。主要表现为新增机器及机器异常重启时。
>
 比如说我们现在有两个机器A和B，则A机器可以用起始值1，步长2（其生成的ID为：1，3，5，7...）；
>
 机器B起始值为2，步长为2（其生成的ID为：2，4，6...），来满足全局唯一性。
>
 如果随着业务增长，需再增加一台机器呢？那我们步长得改为3.起始值得设为一个比现在生成的最大值更大的值，另外两台机器都得做调整。
> 另外需要记录当前机器生成的最大值，在机器异常重启后进行恢复。

### 方案三：

使用号段。

>具体做法是在数据库中存储一个最大值字段MaxValue，然后开始计数，仅当值到达MaxValue后再更新最大值（一种预先分配的思想，仅记录最大值）。
>如：数据库最大值为10000，每次到达后步长+10000，发号机当前值为0，则各业务从发号机取号，每次发号+1，则到达10000后，更新数据库，数据库中最大值变更为20000，发号机器从10000逐渐增加到20000。
>如果各业务想进行隔离，则数据库中可以存储该业务方ID。

### 方案四：

基于雪花算法(Snowflake)

该算法是twitter公司内部分布式项目采用的ID生成算法。

使用了8字节（64位），比MongoID位数少4字节，具体如下：

![](http://imflybird.cn/static/img/2020/Snowflake.png)

其生成的结果为int64。其中第一位保留不用（正数），其余位具体见上图。

其中机器ID最大为：1024，

自增序列号最大为4096，即1毫秒内最多生成4096个数，如果需求超过，需要等待到下一毫秒（qps高达4096000，一般业务基本用不到），可支持使用
2^41/(1000*3600*24*365)=69.7年，且无需借助数据库，可以说性能很高，而且因为其毫秒时间戳设计，如果不是1毫秒内连续生成，则就不会连续。

那雪花算法有什么缺点吗？

那就是时间回拨。雪花算法依赖系统时间，如果出现了时间回拨，则生成的ID就无法保证全局唯一了。

那还有什么更好的方案吗？

### 我的构想

我希望在雪花算法基础上做出这样一款分布式ID生成：

>1.生成ID时不依赖系统时间，这样就不会有时间回拨.
>
>2.性能更高：毕竟每次调用系统时间也是有不少时间消耗。

>3.最小依赖，无需运营。这样在新增机器，或者机器重启时无需人工运营。

于是我设计了这样一款分布式ID生成器-[流星算法](https://github.com/TheFutureIsOurs/meteor)。

先上下其和[雪花算法](https://github.com/TheFutureIsOurs/learncode/blob/master/snow/snow.go)的benchmark对比。

运行机器：联想小新pro13 Ryzen 5 3550H

go版本：1.15

	goos: windows
	goarch: amd64
	BenchmarkSnowflake-8
	4917643	       244 ns/op	       0 B/op	       0 allocs/op
	BenchmarkMeteor-8
	52173231	   22.6 ns/op	       0 B/op	       0 allocs/op

可以看出流星算法比雪花算法快10倍，下面介绍下流星算法。

### 方案五（实现构想）：

[流星算法](https://github.com/TheFutureIsOurs/meteor)

也是64位，

算法组成：

![](http://imflybird.cn/static/img/2020/meteor.png)

设计初衷：

第一位同样保留（正数）

Data位为当前NodeID被创建时的秒级差，这样只在NodeID创建时需要依赖系统时间，后续生成ID时就无需系统时间，就可以防止时间回拨。

NodeID位为节点ID，为了确保生成ID唯一，如果发生了新增机器或服务重启，则NodeID需要每次增加。这样即使发生了时间回拨，由于NodeID唯一，则可以保证最终生成ID唯一性。

自增序列号，11位，最大2048。每次生成，自增序列号+1，当加满后，Data位+1。

随机数位，3位。为什么不把自增序列号和随机数位合成为自增序列号位呢？主要是为了特性4：非连续性。

雪花算法在生成时依赖了毫秒，时间位很细，只有都在这一毫秒内连续生成的ID才会连续，这种条件非常苛刻（qps达到4096000生成才会连续）。

所以加了这个随机数位来保证生成的ID非连续。那随机数如何生成呢？大多数随机数以系统时间作为种子，但是这样就达不到去系统时间的高性能了，我希望一种不依赖系统时间的高效随机数生成算法。最终选用了[Xorshift算法](https://en.wikipedia.org/wiki/Xorshift)。

生成10个ID如下：

	5016762319896585
	5016762319896596
	5016762319896600
	5016762319896614
	5016762319896616
	5016762319896626
	5016762319896635
	5016762319896640
	5016762319896651
	5016762319896656

### 总结

总结下来就是，
>1.NodeID创建时依赖当前系统时间，但是生成时不需系统时间来达到去系统时间化，这样就解决了时间回拨。

>2.NodeID每次需要跟以前不重复。这样就保证了全局唯一。

>3.随机数位保证了生成ID非连续。

在使用时可以使用Mysql自增ID来注册NodeID。






