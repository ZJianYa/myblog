# 概述

导出数据的数据，或者备份的时候，数据库肯定是在全表扫描

假设，我们现在要对一个 200G 的 InnoDB 表 db1. t，执行一个全表扫描。当然，你要把扫描结果保存在客户端，会使用类似这样的命令：

mysql -h$host -P$port -u$user -p$pwd -e "select * from db1.t" > $target_file

## 结果集

那么结果集存在哪里？实际上数据库肯定是分批加载

1. 结果存在 net_buffer 中，大小由 net_buffer_length 定义  
  写满时，发送出去  
2. 发送成功，则清空并重新写入新数据
3. 如果发送函数返回 EAGAIN 或 WSAEWOULDBLOCK，就表示本地网络栈（socket send buffer）写满了，进入等待。  
  直到网络栈重新可写，再继续发送。

当然这个发送过程会受到客户端的获取速度限制。

对于正常的线上业务来说，如果一个查询的返回结果不会很多的话，我都建议你使用 mysql_store_result 这个接口，直接把查询结果保存到本地内存。  
有同学说到自己因为执行了一个大查询导致客户端占用内存近 20G，这种情况下就需要改用 mysql_use_result 接口了。  

另一方面，如果你在自己负责维护的 MySQL 里看到很多个线程都处于“Sending to client”这个状态，  
就意味着你要让业务开发同学优化查询结果，并评估这么多的返回结果是否合理。

与“Sending to client”长相很类似的一个状态是“Sending data”，这是一个经常被误会的问题。  
有同学问我说，在自己维护的实例上看到很多查询语句的状态是“Sending data”，但查看网络也没什么问题啊，为什么 Sending data 要这么久？

实际上，一个查询语句的状态变化是这样的（注意：这里，我略去了其他无关的状态）：

<ul>
<li>MySQL 查询语句进入执行阶段后，首先把状态设置成“Sending data”；</li>
<li>然后，发送执行结果的列相关的信息（meta data) 给客户端；</li>
<li>再继续执行语句的流程；</li>
<li>执行完成后，把状态设置成空字符串。</li>
</ul>

也就是说，“Sending data”并不一定是指“正在发送数据”，而可能是处于执行器过程中的任意阶段。  
比如，你可以构造一个锁等待的场景，就能看到 Sending data 状态。  


## 全表扫描

在第 2 篇文章的评论区有同学问道，由于有 WAL 机制，当事务提交的时候，磁盘上的数据页是旧的，那如果这时候马上有一个查询要来读这个数据页，是不是要马上把 redo log 应用到数据页呢？

答案是不需要。因为这时候内存数据页的结果是最新的，直接读内存页就可以了。  
你看，这时候查询根本不需要读磁盘，直接从内存拿结果，速度是很快的。所以说，Buffer Pool 还有加速查询的作用。

而 Buffer Pool 对查询的加速效果，依赖于一个重要的指标，即：内存命中率

你可以在 show engine innodb status 结果中，查看一个系统当前的 BP 命中率。  
一般情况下，一个稳定服务的线上系统，要保证响应时间符合要求的话，内存命中率要在 99% 以上。  

执行 show engine innodb status ，可以看到“Buffer pool hit rate”字样，显示的就是当前的命中率。比如图 5 这个命中率，就是 99.0%。

## 