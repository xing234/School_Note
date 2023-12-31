# MySQL事务

https://juejin.cn/post/7157956679932313608

[21 数据库备份：备份文件也要检查！ (lianglianglee.com)](https://learn.lianglianglee.com/专栏/MySQL实战宝典/21  数据库备份：备份文件也要检查！.md)

全解MySQL中的各类日志，如撤销日志、错误日志、慢查询日志、中继日志、回滚日志.....

而MySQL中，很多很多的功能也都需要基于日志实现，比如事务回滚、数据持久化、数据恢复、数据迁移、MVCC机制.....

## 一、Undo-log撤销日志

Undo即撤销的意思，为回滚日志，用来给MySQL撤销SQL操作的。

当一条写入类型的SQL执行时，都会记录Undo-log日志，会生成相应的反SQL放入到Undo-log中，例如：

```
·如果目前是insert插入操作，则生成一个对应的delete操作。

·如果目前是delete删除操作，InnoDB中会修改隐藏字段deleted_bit=1，则生成改为0的语句。

·如果目前的update修改操作，比如将姓名从竹子改成了熊猫，那就生成一个从熊猫改回竹子的操作。
```

当事务中某条SQL执行失败时，MySQL就需要回滚事务中其他执行成功的SQL，此时就会找到这个事务在Undo-log中生成的反SQL，然后将库中的数据改回事务发生前的样子。

但其实这种说法是错误的，**Undo-log并不存在单独的日志文件**，也就是磁盘中并不会存在xx-undo.log这类的文件。InnoDB默认是将Undo-log存储在xx.ibdata共享表数据文件当中，默认采用段的形式存储。

当一个事务尝试写某行表数据时，首先会将旧数据拷贝到**xx.ibdata**文件中，将表中行数据的隐藏字段：**roll_ptr回滚指针**会指向xx.ibdata文件中的旧数据，然后再写表上的数据。

在共享表数据文件中，有一块区域名为Rollback Segment回滚段，每个回滚段中有1024个Undo-log Segment，每个Undo段可存储一条旧数据，而执行写SQL时，Undo-log就是写入到这些段中。

不过在`MySQL5.5`版本前，默认只有一个`Rollback Segment`，而在`MySQL5.5`版本后，默认有`128`个回滚段，即支持`128*1024`条`Undo`记录同时存在。

### 1.1、对于事务回滚原理的纠正

实际上当一个事务需要回滚时，本质上并不会以执行反`SQL`的模式还原数据，而是直接将`roll_ptr`回滚指针指向的`Undo`记录，从`xx.ibdata`共享表数据文件中拷贝到`xx.ibd`表数据文件，覆盖掉原本改动过的数据。

![avatar](figures\1.webp)

一条写`SQL`执行的流程如上图中的序号所示，当需要回滚事务时，直接用`Undo`旧记录覆盖表中修改过的新记录即可！

如果是`insert`操作，由于插入之前这条数据都不存在，那么就不会产生`Undo`记录，此时回滚时如何删除这条记录呢？因为插入操作不会产生`Undo`旧记录，因此隐藏字段中的`roll_ptr=null`，因此直接用`null`覆盖插入的新记录即可，这样也就实现了删除数据的效果~

### 1.2 基于Undo版本链实现MVCC

`Undo-log`中记录的旧数据并不仅仅只有一条，一条相同的行数据可能存在多条不同版本的`Undo`记录，内部会通过`roll_ptr`回滚指针，组成一个单向链表，而这个链表则被称之为`Undo`版本链，案例如下：

```mysql
-- 事务T1：trx_id=1（两次修改同一条数据）
UPDATE `zz_users` SET user_name = "竹子" WHERE user_id = 1;
UPDATE `zz_users` SET user_sex = "男" WHERE user_id = 1;
```

`Undo-log`中的旧数据版本链示意图大致如下：

<img src="figures\2.webp" alt="avatar" style="zoom:150%;" />

### 1.3 Undo-log的内存缓冲区

`InnoDB`在`MySQL`启动时，会在内存中构建一个`BufferPool`，而这个缓冲池主要存放两类东西，一类是数据相关的缓冲，如索引、锁、表数据等，另一类则是各种日志的缓冲，如`Undo、Bin、Redo....`等日志。

而当一条写`SQL`执行时，不会直接去往磁盘中的`xx.ibdata`文件写数据，而是会写在`undo_log_buffer`缓冲区中，因为工作线程直接去写磁盘太影响效率了，写进缓冲区后会由后台线程去刷写磁盘。

```
那么如果当一个事务提交时，Undo的旧记录会不会立马被删除呢？因为事务都提交了，不需要再回滚改动过的数据，似乎用不上Undo旧记录了，对吗？确实如此，但不会立马删除Undo记录，对于旧记录的删除工作，InnoDB中会有专门的purger线程负责，purger线程内部会维护一个ReadView，它会以此作为判断依据，来决定何时移除Undo记录。
```

为什么不是事务提交后立马删除`Undo`记录呢？因为可能会有其他事务在通过快照，读`Undo`版本链中的旧数据，直接移除可能会导致其他事务读不到数据，因此删除的工作就交给了`purger`线程。

### 1.4 Undo-log相关的参数（好像没啥用）

最后再来看看关于`Undo-log`的一些参数，其实在`MySQL5.5`之前没有太多参数，如下：

- `innodb_max_undo_log_size`：本地磁盘文件中，`Undo-log`的最大值，默认`1GB`。
- `innodb_rollback_segments`：指定回滚段的数量，默认为`1`个。

除开上述两个参数外，其他参数基本上是在`MySQL5.6`才有的，如下：

- `innodb_undo_directory`：指定`Undo-log`的存放目录，默认放在`.ibdata`文件中。
- `innodb_undo_logs`：指定回滚段的数量，默认为`128`个，也就是之前的`innodb_rollback_segments`。
- `innodb_undo_tablespaces`：指定`Undo-log`分成几个文件来存储，必须开启`innodb_undo_directory`参数。
- `innodb_undo_log_truncate`：是否开启`Undo-log`的在线压缩功能，即日志文件超过大小一半时自动压缩，默认`OFF`关闭。

没错，在`MySQL5.5`版本以后，`Undo-log`日志支持单独存放，并且多出了几个参数可以调整`Undo-log`的区域。

## 二、Redo-log重做日志

这两日志都是`InnoDB`引擎独有的，`Undo-log`主要用于实现事务回滚和`MVCC`机制，而`Redo-log`则用来实现数据的恢复。还记得在[《MySQL事务篇》](https://juejin.cn/post/7152765784299667487)中聊到的数据恢复机制嘛？

![avatar](figures\3.webp)

### 2.1 为何需要Redo-log日志？

`MySQL`绝大部分引擎都是是基于磁盘存储数据的，但如若每次读写数据都走磁盘，其效率必然十分低下，因此`InnoDB`引擎在设计时，当`MySQL`启动后就会在内存中创建一个`**BufferPool**`，运行过程中会将大量操作汇集在内存中进行，比如写入数据时，先写到内存中，然后由后台线程再刷写到磁盘。

```
虽然使用`BufferPool`提升了`MySQL`整体的读写性能，但它是基于内存的，也就意味着随着机器的宕机、重启，其中保存的数据会消失，那当一个事务向内存中写入数据后，`MySQL`突然宕机了，代表这条未刷写到磁盘的数据会丢失。因此使用Redo-log
```

因为数据写到内存后有丢失风险，这明显违背了事务`ACID`原则中的持久性，所以`Redo-log`的出现就是为了解决该问题，`Redo-log`是一种预写式日志，即在向内存写入数据前，会先写日志，当后续数据未被刷写到磁盘、`MySQL`崩溃时，就可以通过日志来恢复数据，确保所有提交的事务都会被持久化。

```
工作线程执行`SQL`前，写的`Redo-log`日志，也是写在了内存中的`redo_log_buffer`缓冲区。
```

既然`Redo-log`日志也是先写内存，那`Redo-log`有没有丢失的风险呢？这跟`Redo-log`的刷盘策略有关。

### 2.2 Redo-log的刷盘策略

对于内存中的`redo_log_buffer`缓冲区，其中写入的数据会何时被刷写到磁盘？对于这点在之前[《SQL执行篇-写SQL执行时的日志操作》](https://juejin.cn/post/7145102393988874253#heading-13)中简单的提到过：

![avadar](figures\4.webp)

简单来说就是刷盘的时机由`innodb_flush_log_at_trx_commit`参数来控制，默认是处于第二个级别，也就是每次提交事务时都会刷盘，这也就意味着一个事务执行成功后，相应的`Redo-log`日志绝对会被刷写到磁盘中，因此无需担心会出现丢失风险。

```
那如果事务还未提交时，MySQL宕机怎么办？对于这个问题在事务篇的那个截图中有，不再反复赘述！
```

但再来思考一个问题：既然`Redo-log`要写磁盘，那为何不在写日志的时候，直接把数据写到磁盘里面去呢？

### 2.3 Redo-log为何“多此一举”

先刷写一次`Redo-log`日志到磁盘，后台线程再根据`Redo-log`日志把数据落盘，这个动作似乎看起来有些多余对吧？但实际上这样做好处很大：

- ①日志比数据先落入磁盘，因此就算`MySQL`崩溃也可以通过日志恢复数据。
- ②写日志时是以追加形式写到末尾，而写数据时则是计算数据位置，随机插入。

对于第一点好处就不多说了，重点来聊一聊第二点，因为写日志的时候，只需要将记录追加到日志文件的尾部即可，这是按顺序写入，但写入表数据时，还需要先先计算数据的位置，比如修改一条数据时，需要先判断这条数据在磁盘文件中的那个位置，找到了位置再写入，这是随机写入，顺序写入的速度会比随机写入快很多很多。

2.4 Redo-log相关参数

这里也列举出几个`Redo-log`日志中，较为重要的系统参数：

- `innodb_flush_log_at_trx_commit`：设置`redo_log_buffer`的刷盘策略，默认每次提交事务都刷盘。
- `innodb_log_group_home_dir`：指定`redo-log`日志文件的保存路径，默认为`./`。
- `innodb_log_buffer_size`：指定`redo_log_buffer`缓冲区的大小，默认为`16MB`。
- `innodb_log_files_in_group`：指定`redo`日志的磁盘文件个数，默认为`2`个。
- `innodb_log_file_size`：指定`redo`日志的每个磁盘文件的大小限制，默认为`48MB`。

其中主要讲一下`Redo-log`的本地磁盘文件个数，为啥默认是两个呢？因为`MySQL`通过来回写这两个文件的形式记录`Redo-log`日志，用两个日志文件组成一个“环形”，如下：

![avadar](figures\5.webp)

先来简单解释一下图中存在的两根指针：

- `write pos`：这根指针用来表示当前`Redo-log`文件写到了哪个位置。
- `check point`：这根指针表示目前哪些`Redo-log`记录已经失效且可以被擦除（覆盖）。

两根指针中间区域，也就是图中的红色区域，代表是可以写入日志记录的可用空间，而蓝色区域则表示日志落盘但数据还未落盘的记录，这句话怎么理解呢？

```
当一个事务写了`redo-log`日志、并将数据写入缓冲区后，但数据还未写到本地的表数据文件中，此时这个事务对应的`redo-log`记录就为上图中的蓝色，而当一个事务所写的数据也落盘后，对应的`redo-log`记录就会变为红色。
```

当`write pos`指针追上`check point`指针时，红色区域就会消失，也就代表`Redo-log`文件满了，再当`MySQL`执行写操作时就会被阻塞，因为无法再写入`redo-log`日志了，所以会触发`checkpoint`刷盘机制，将`redo-log`记录对应的事务数据，全部刷写到磁盘中的表数据文件后，阻塞的写事务才能继续执行。

```
`触发checkpoint刷盘机制后，随着数据的落盘，check point指针也会不断的向后移动，红色区域也会不断增长，因此阻塞的写事务才能继续执行。`
```

`checkpoint`机制的系统参数：

- `innodb_log_write_ahead_size`：设置`checkpoint`刷盘机制每次落盘动作的大小，默认为`8K`，如果你要设置，必须要为`4k`的整数倍，这跟`read-on-write`问题有关，具体的可以参考：[《这篇文章》](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_27028561%2Farticle%2Fdetails%2F116540923)。
- `innodb_log_compressed_pages`：是否对`Redo`日志开启页压缩机制，默认`ON`，这跟`InnoDB`的页压缩技术有关，后续《特性篇》聊。
- `innodb_log_checksums`：`Redo`日志完整性效验机制，默认开启，必须要开启，否则有可能刷写数据时，只刷一半，出现类似于“网络粘包”的问题。

后续几个参数略微有些复杂，因为主要跟`MySQL5.6`之后的优化有关，后续在《MySQL特性篇》中会再次细聊。

## 三、Bin-log变更日志

`Bin-log`日志也被称之为二进制日志，作用与`Redo-log`类似，主要是记录所有对数据库表结构变更和表数据修改的操作，对于`select、show`这类读操作并不会记录。`bin-log`是`MySQL-Server`级别的日志，也就是所有引擎都能用的日志，而`redo-log、undo-log`都是`InnoDB`引擎专享的，无法跨引擎生效。

![avadar](C:\XingMinYu\java后端\biji\figures\6.webp)

看到这张写`SQL`的执行流程图，重点观察里面的第`⑨`步，无论当前表使用的是什么引擎，实际上都需要完成记录`bin-log`日志这步操作，和之前分析的两种日志相同，`bin-log`也由内存日志缓冲区+本地磁盘文件两部分组成，这也就意味着：写`bin-log`日志时，也会先写缓冲区，然后由后台线程去刷盘。

### 3.1 bin-log的缓冲区

bin-log跟`redo-log、undo-log`的缓冲区并不同，前面两种日志缓冲区都位于`InnoDB`创建的共享`BufferPool`中，而`bin_log_buffer`是位于每条线程中的，关系图如下：

![avadar](C:\XingMinYu\java后端\biji\figures\7.webp)

`MySQL-Server`会给每一条工作线程，都分配一个`bin_log_buffer`，而并不是放在共享缓冲区中，这是为啥呢？因为`MySQL`设计时要兼容所有引擎，直接将`bin-log`的缓冲区，设计在线程的工作内存中，这样就能够让所有引擎通用，并且不同线程/事务之间，由于写的都是自己工作内存中的`bin-log`缓冲，因此并发执行时也不会冲突！

`bin_log_buffer`的设计，就类似于咱们之前讲[《并发编程》](https://juejin.cn/post/7041073881188139039)时讲过的[《ThreadLocal线程变量副本》](https://juejin.cn/post/7038470209262321695)。

简单理解`bin-log`缓冲区的设计后，对于`bin-log`的刷盘策略就不反复赘述了，就是通过`sync_binlog`参数控制，与之前`redo-log`类似（上面有）。

### 3.2 Bin-log本地日志文件的格式

`bin-log`的本地日志文件，采用的是追加写的模式，也就是一直向文件末尾写入新的日志记录，当一个日志文件写满后，会创建一个新的`bin-log`日志文件，每个日志文件的命名为`mysql-bin.000001、mysql-bin.000002、mysql-bin.00000x....`，可以通过`show binary logs;`命令查看已有的`bin-log`日志文件。

```
接着再来聊聊`bin-log`文件的内部格式~
```

在`bin-log`的本地文件中，其中存储的日志记录共有`Statment、Row、Mixed`三种格式，分别是啥意思呢？

```
`Statment`：每一条会对数据库产生变更的`SQL`语句都会记录到`bin-log`中。
```

举个例子：

```mysql
-- 查询一次用户表数据，如下：
SELECT * FROM `zz_users`;
+---------+-----------+----------+----------+---------------------+
| user_id | user_name | user_sex | password | register_time       |
+---------+-----------+----------+----------+---------------------+
|       1 | 熊猫      | 女       | 6666     | 2022-08-14 15:22:01 |
|       2 | 竹子      | 男       | 1234     | 2022-09-14 16:17:44 |
|       3 | 子竹      | 男       | 4321     | 2022-09-16 07:42:21 |
|       4 | 猫熊      | 女       | 8888     | 2022-09-27 17:22:59 |
|       9 | 黑竹      | 男       | 9999     | 2022-09-28 22:31:44 |
+---------+-----------+----------+----------+---------------------+

-- 将用户表中所有 ID>3的密码重置
update `zz_users` set `password` = "1111" where user_id > 3;

```

比如上述这个事务执行时，`MySQL`会将第二条`update`语句记录在`bin-log`日志中，但对于`select`语句则不会记录（在记录`SQL`时，还会记录一下`SQL`的上下文信息，如执行时间、事务ID、日志量......）。

这种方式的优势很明显，由于只记录对数据库产生变更操作的`SQL`，所以不会产生太大的日志量，节约空间，恢复数据时因为数据量小，所以磁盘`IO`次数少，因此性能会比较不错。同时做主备等高可用架构时，数据同步也会较小，因此比较节省带宽。

```
但虽然优势不小，但缺点页很明显，即恢复数据、主从同步数据时，有时会出现数据不一致的情况，如`SQL`中使用了`sysdate()、now()`这类函数，比如举个简单的例子：
```

```mysql
insert into `zz_users` values(11,"棕熊","男","3333",sysdate());
```

比如这条插入语句，由于对用户表产生了变更操作，所以会被记录到`bin-log`中，但当主从架构之间做数据同步时，假设将这条`SQL`同步到从机上执行，此时问题就来了，`sysdate()`函数会获取机器的当前时间，但主机和从机执行这条`SQL`显然不是同一时间，因此就会导致`ID=11`的这条数据，在主机和从机的用户表中，注册时间会出现不一致。

```
`Row`：这种模式就是为了解决`Statment`模式的缺陷，`Row`模式中不再记录每条造成变更的`SQL`语句，而是记录具体哪一个分区中的、哪一个页中的、哪一行数据被修改了。
```

这又怎么理解呢？还是以前面的重置密码的例子来说：

```mysql
-- 将用户表中所有 ID>3的密码重置（ID=4、9的两条数据会被重置）
update `zz_users` set `password` = "1111" where user_id > 3;
```

在这种模式下，就不会记录这条`update`语句，而是记录发生改变的行数据，即`ID=4、9`的两条用户数据，会将其更改后的值记录到`bin-log`日志中。

这种方式因为不记录`SQL`，而是记录修改后的值，因此有个很大的好处是：当主从同步数据时，复制的是主机上的数据，因此不会出现主从数据不一致的情况。但缺陷同样很明显，比如表中有`800W`数据，现在我对`ID<600W`的所有数据进行了修改操作，哪也就意味着会有`600W`条记录写入`bin-log`日志，这个数据量可想而知，其磁盘`IO`、网络带宽开销会很高。

```
`Mixed`：这种被称为混合模式，即`Statment、Row`的结合版，因为`Statment`模式会导致数据出现不一致，而`Row`模式数据量又会很大，因此`Mixed`模式结合了两者的优劣势，对于可以复制的`SQL`采用`Statment`模式记录，对于无法复制的`SQL`采用`Row`记录。
```

这样即保留了`Statment`模式的数据量小，又具备`Row`模式的数据精准性，鱼和熊掌兼得焉~

其实看到这里，如果比较熟悉`Redis4.x`版本的小伙伴应该会有种熟悉感，`Redis`的`RDB、AOF`持久化模式，正好对应`MySQL`的`Statment、Row`模式，而`Redis4.0`引入了混合持久化机制，`MySQL5.1`版本也引入了混合日志模式~

### 3.3 为什么有了Bin-log还需要Redo-log？

`Redo-log、Bin-log`都是记录更新数据库的操作，但为什么会同时设计两个呢？这其实跟`InnoDB`有关，如若对`MySQL`旧史较为熟悉的小伙伴应该知道，`MySQL`自己的官方引擎实际上最初是`MyISAM`，`InnoDB`是`Innobase-Oy`公司开发的一款可拔插式引擎，由于`InnoDB`被`MySQL`支持后使用频率越来越高，后面`MySQL`官方才用`InnoDB`替换了`MyISAM`作为默认引擎。

```
那这跟上面的问题有啥关系呢？其实关系很大，MySQL-Server、MyISAM是出自于官方的产品，因此MyISAM中并未设计记录变更操作的日志，记录变更操作由MySQL-Server来通过Bin-log完成。
```

但因为`MyISAM`不支持事务，所以`MySQL-Server`设计的`Bin-log`无法用于灾难恢复，因此`InnoDB`在设计时，又重新设计出`Redo-log`日志，可以利用该日志实现`crash-safe`灾难恢复能力，确保任何事务提交后数据都不会丢失。

3.4 Redo-log、Bin-log两者的区别

对于`Redo-log、Bin-log`两者的区别，主要可以从四个维度上来说：

- ①生效范围不同，`Redo-log`是`InnoDB`专享的，`Bin-log`是所有引擎通用的。
- ②写入方式不同，`Redo-log`是用两个文件循环写，而`Bin-log`是不断创建新文件追加写。
- ③文件格式不同，`Redo-log`中记录的都是变更后的数据，而`Bin-log`会记录变更`SQL`语句。
- ④使用场景不同，`Redo-log`主要实现故障情况下的数据恢复，`Bin-log`则用于数据灾备、同步。

### 3.5 不小心删库后应该跑路吗

首先来说一下，为啥要讨论这个问题呢，这是由于之前[《MySQL架构篇》](https://juejin.cn/post/7143614079532269598)的评论区的一位小伙伴提出的：

![avadar](figures\8.webp)

这里有两个问题：①删库后跑路会不会被人发现？②`MySQL`能不能和`Oracle`一样具备闪回功能？

先来简单聊聊第一个问题，如果你在线上真的删库了，哪就先别想着跑路，你跑不掉！因为`bin-log`日志中会记录执行`SQL`的连接会话信息，同时一般规模较大的企业，都会搭建完善的监控系统，会监控服务的网络连接，因此当你删库后，可以顺着`bin-log → session → network-connection`这条线确定执行删库`SQL`的`IP`！如果你还未断开连接，直接通过`MySQL`的命令就能定位到删库的`IP`，因此基本上删库了，是可以定位到责任人的！

当然，如果项目配备的监控系统不够完善，同时你的连接已经断开，并且电脑换了一个局域网，同时时间来到了三天以后，如果还没人发现你，哪基本上跑路也不会有人发现，但这样干，会存在些许做贼心虚的嫌疑~

```
误删了数据就要想办法恢复，咋恢复呢？通过日志恢复，但`Redo-log、Bin-log`都会记录数据库的变更操作，因此用谁比较合适呢？
```

答案是`Bin-log`，因为`Redo-log`采用循环写的方式，一边写会一边擦，里面无法得到完整的数据，而`Bin-log`是追加写的模式，你不去主动删除磁盘的日志文件，并且磁盘的空间还足够，一般`Bin-log`日志文件都会在本地，因此当你删库后，可以直接去本地找`Bin-log`的日志文件，然后拷贝出来一份，再打开最后一个文件，把里面删库的记录手动移除，再利用`mysqlbinlog`工具导出`xx.SQL`文件，最后执行该`SQL`文件即可恢复删库前的数据。

```
这里就叙说大体逻辑，具体的数据恢复操作，会在后续的《MySQL线上排查与数据恢复篇》细讲，其实也可以通过`Flashback`工具提供的闪回功能恢复数据，但以后再细聊~
```

### 3.6 bin-log相关的参数

`log_bin`：是否开启`bin-log`日志，默认`ON`开启，表示会记录变更`DB`的操作。

`log_bin_basename`：设置`bin-log`日志的存储目录和文件名前缀，默认为`./bin.0000x`。

`log_bin_index`：设置`bin-log`索引文件的存储位置，因为本地有多个日志文件，需要用索引来确定目前该操作的日志文件。

`binlog_format`：指定`bin-log`日志记录的存储方式，可选`Statment、Row、Mixed`。

`max_binlog_size`：设置`bin-log`本地单个文件的最大限制，最多只能调整到`1GB`。

`binlog_cache_size`：设置为每条线程的工作内存，分配多大的`bin-log`缓冲区。

`sync_binlog`：控制`bin-log`日志的刷盘频率。

`binlog_do_db`：设置后，只会收集指定库的`bin-log`日志，默认所有库都会记录。

### 3.7 Redo-log的两阶段提交

`MySQL`事务两阶段提交方案，是指`Redo-log`分两次写入，如下：

![avadar](figures\9.webp)



注意看之前给出的写`SQL`执行流程图，其中第⑤、⑩步，分别会写两次`Redo-log`日志，这个日志的作用前面讲的很明白了，主要用来做崩溃恢复，但为什么要分两次写呢？写一次不行嘛？

其实想要弄明白这个问题，要结合`bin-log`日志一起来聊。

如果只写一次的话，那到底先写`bin-log`还是`redo-log`呢？

```
先写`bin-log`，再写`redo-log`：当事务提交后，先写`bin-log`成功，结果在写`redo-log`时断电宕机了，再重启后由于`redo-log`中没有该事务的日志记录，因此不会恢复该事务提交的数据。但要注意，主从架构中同步数据是使用`bin-log`来实现的，而宕机前`bin-log`写入成功了，就代表这个事务提交的数据会被同步到从机，也就意味着从机会比主机多出一条数据。
```

```
先写`redo-log`，再写`bin-log`：当事务提交后，先写`redo-log`成功，但在写`bin-log`时宕机了，主节点重启后，会根据`redo-log`恢复数据，但从机依旧是依赖`bin-log`来同步数据的，因此从机无法将这个事务提交的数据同步过去，毕竟`bin-log`中没有撒，最终从机会比主机少一条数据。
```

经过上述分析后可得知：如果`redo-log`只写一次，那不管谁先写，都有可能造成主从同步数据时的不一致问题出现，为了解决该问题，`redo-log`就被设计成了两阶段提交模式，设置成两阶段提交后，整个执行过程有三处崩溃点：

- `redo-log(prepare)`：在写入准备状态的`redo`记录时宕机，事务还未提交，不会影响一致性。
- `bin-log`：在写`bin`记录时崩溃，重启后会根据`redo`记录中的事务`ID`，回滚前面已写入的数据。
- `redo-log(commit)`：在`bin-log`写入成功后，写`redo(commit)`记录时崩溃，因为`bin-log`中已经写入成功了，所以从机也可以同步数据，因此重启时直接再次提交事务，写入一条`redo(commit)`记录即可。

通过这种两阶段提交的方案，就能够确保`redo-log、bin-log`两者的日志数据是相同的，`bin-log`中有的主机再恢复，如果`bin-log`没有则直接回滚主机上写入的数据，确保整个数据库系统的数据一致性。

```
为什么`bin-log`又被叫做二进制日志呢？因为记录日志时，`MySQL`写入的是二进制数据，而并非字符数据，也就意味着直接用`cat/vim`这类工具是无法打开的，必须要通过`MySQL`提供的`mysqlbinlog`工具解析查看。
```

## 四、Error-log错误日志

前面已经将最重要的`undo-log、redo-log、bin-log`三大日志讲明白了，这三个日志都是用来辅助`MySQL、InnoDB`在线上正常运行的，但凡其中一个出现问题，都有可能导致`MySQL`无法正常工作。

```
接下来再看几个辅助性的日志，即`error-log、slow-log、relay-log`。
```

`error-log`：`MySQL`线上`MySQL`由于非外在因素（断电、硬件损坏...）导致崩溃时，辅助线上排错的日志。

`slow-log`：系统响应缓慢时，用于定位问题`SQL`的日志，其中记录了查询时间较长的`SQL`。

`relay-log`：搭建`MySQL`高可用热备架构时，用于同步数据的辅助日志。

接下来先看`error-log`，这个日志的作用很明显，从名字都能得知它是用于记录`MySQL`报错信息的，其中涵盖了`MySQL-Server`的启动、停止运行的时间，以及报错的诊断信息，也包括了错误、警告和提示等多个级别的日志详情。

```
通过错误日志，一方面可以用来监控`MySQL`的运行状态，便于预防故障、发现故障，同时也可以在出现问题时，用来辅助排查问题、修复故障，因为`MySQL-Server`的错误日志是默认开启的，并且无法手动关闭！
```

一般来说，`error-log`日志文件默认是在`MySQL`安装目录下的`data`文件夹中，但如果你想要改变位置，哪也可以通过`log-error`这个参数，来手动指定保存的位置与文件名。

```
如果你不清楚错误日志的位置，也可以通过`SHOW VARIABLES LIKE 'log_error';`命令来查看。
```

最后稍微提一嘴，如何根据错误日志来排错问题呢？实际上非常简单，在`MySQL`故障的情况下，打开`error-log`文件，然后搜索`Error、Waiting`级别的日志记录，然后参考诊断信息即可。

## 五、Slow-log慢查询日志

对于线上响应缓慢的问题，一步步的排查过程之后还未找到问题，最终就会来到数据库，尝试对`SQL`或索引调优，但一个项目中，存在成千上万条`SQL`，到底是由于哪条`SQL`造成的响应缓慢，如果一条条去分析，其工作量定然非常吃力，为了排查问题时足够轻松，`MySQL`官方支持开启慢查询日志。

慢查询日志是什么呢？也就是当一条`SQL`执行的时间超过规定的阈值后，那么这些耗时的`SQL`就会被记录在慢查询日志中，当线下出现响应缓慢的问题时，可以直接通过查看慢查询日志定位问题，定位到产生问题的`SQL`后，再用`explain`这类工具去生成`SQL`的执行计划，然后根据生成的执行计划来判断为什么耗时长，是由于没走索引，还是索引失效等情况导致的。

```
不过对于慢查询`SQL`的监控，`MySQL`默认是关闭的，也就是说`MySQL`默认不会记录慢查询日志，因为为了后续线上问题好排查，项目上线前一定要记得开启！
```

- `slow_query_log`：设置是否开启慢查询日志，默认`OFF`关闭。
- `slow_query_log_file`：指定慢查询日志的存储目录及文件名。

可以通过这两个参数来开启慢查询日志，如果不设置存储目录，默认放在`MySQL`的具体库的目录下。当开启慢查询日志的监控后，可以通过设置`long_query_time`参数，来指定查询`SQL`的阈值：

```mysql
set global long_query_time = 1;
```

其默认单位是秒，因此如果要指定更细粒度的时间，可以通过`0.01`这种形式设置，`0.01`表示`10ms`。当然，该参数也可不设置，不指定阈值的情况下，默认为`10s`，即执行时间超过`10s`的查询`SQL`才会记录到慢查询日志中。

```
对于阈值的设置，并不是随咱们率性而为，这个参数一定要设置合理！因为该参数的大小会直接影响`MySQL`的性能，比如设置一个`0.2s`，但如果大量业务`SQL`执行时都会超出该时长，那最终会导致`MySQL`十分频繁的往慢查询日志中写数据。
```

要记住：慢查询日志在内存中是没有缓冲区的，也就意味着每次记录慢查询`SQL`，都必须触发磁盘`IO`来完成，因此阈值设的太小，容易使得`MySQL`性能下降；如果设的太大，又会导致无法检测到问题`SQL`，因此该值一定要设置一个合理值。

```
问题来了，这个值设成多大合理呢？可以先开启`general log`，观察后实际的业务情况后再决定。
```

### General-log查询日志

`general log`即查询日志，`MySQL`会向其中写入所有收到的查询命令，如`select、show`等，同时要注意：无论`SQL`的语法正确还是错误、也无论`SQL`执行成功还是失败，`MySQL`都会将其记录下来。对于该日志可以通过下述参数开启：

- `general_log`：是否开启查询日志，默认`OFF`关闭。
- `general_log_file`：指定查询日志的存储路径和文件名（默认在库的目录下，主机名+`.log`）。

项目测试阶段，可以先开启查询日志，然后压测所有业务，紧接着再分析日志中`SQL`的平均耗时，再根据正常的`SQL`执行时间，设置一个偏大的慢查询阈值即可（这是个笨办法，如果项目规模较大，直接设置一个大概值，然后上灰度发布，走正式的运营场景效果会更佳）。

```
当然，压测阶段结束后，项目正式上线前，一定要记得关闭普通查询日志！！
```

## 六、Relay-log中继日志

 `relay log`在单库中是见不到的，该类型的日志仅存在主从架构中的从机上，主从架构中的从机，其数据基本上都是复制主机`bin-log`日志同步过来的，而从主机复制过来的`bin-log`数据放在哪儿呢？也就是放在`relay-log`日志中，中继日志的作用就跟它的名字一样，仅仅只是作为主从同步数据的“中转站”。

当主机的增量数据被复制到中继日志后，从机的线程会不断从`relay-log`日志中读取数据并更新自身的数据，`relay-log`的结构和`bin-log`一模一样，同样存在一个`xx-relaybin.index`索引文件，以及多个`xx-relaybin.00001、xx-relaybin.00002....`数据文件。

```
对于这个日志的具体参数、工作过程，放在后续的《MySQL高可用-主从读写分离篇》阐述。
```

## 七、数据库备份

对架构师来说，还要做好备份架构的设计。因为我们要防范意外情况的发生，比如黑客删除了数据库中所有的核心数据；又或者某个员工有意也罢、无意也好，删除了线上的数据。

### 7.1 数据库备份

复制技术（Replication）或 InnoDB Cluster 只负责业务的可用性，保障数据安全除了线上的副本数据库，我们还要构建一个完整的离线备份体系。这样即使线上数据库被全部破坏，用户也可以从离线备份恢复出数据。

所以，第一步要做好：**线上数据库与离线备份系统的权限隔离**。

也就是说，可以访问线上数据库权限的同学一定不能访问离线备份系统，反之亦然。否则，如果两边的数据都遭受破坏，依然无法恢复数据。

而对于 MySQL 数据库来说，数据库备份分为全量备份、增量备份。

#### 全量备份

指备份当前时间点数据库中的所有数据，根据备份内容的不同，**全量备份可以分为逻辑备份、物理备份两种方式。**

##### **逻辑备份**

指备份数据库的逻辑内容，就是每张表中的内容通过 INSERT 语句的形式进行备份。

MySQL 官方提供的逻辑备份工具有 mysqldump 和 mysqlpump。通过 mysqldump 进行备份，可以使用以下 SQL 语句：

```mysql
mysqldump -**A** --single-transaction > backup.sql
```

上面的命令就是通过 mysqldump 进行全量的逻辑备份：

1. 参数 -A 表示备份所有数据库；
2. 参数 –single-transaction 表示进行一致性的备份。

我特别强调，参数 –single-transaction 是必须加的参数，否则备份文件的内容不一致，这样的备份几乎没有意义。

如果你总忘记参数 –single-transaction，可以在 MySQL 的配置文件中加上如下提示：

```mysql
# my.cnf 
[mysqldump]
single-transaction
```

按上面配置，每当在服务器上运行命令时 mysqldump 就会自动加上参数 –single-transaction，你也就不会再忘记了。

在上面的命令中，最终的备份文件名为 backup.sql，打开这个文件，我们会看到类似的内容：

```
-- MySQL dump 10.13  Distrib 8.0.23, for Linux (x86_64)

--

-- Host: localhost    Database:

-- ------------------------------------------------------

-- Server version       8.0.23

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;

/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;

/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;

/*!50503 SET NAMES utf8mb4 */;

/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;

/*!40103 SET TIME_ZONE='+00:00' */;

/*!50606 SET @OLD_INNODB_STATS_AUTO_RECALC=@@INNODB_STATS_AUTO_RECALC */;

/*!50606 SET GLOBAL INNODB_STATS_AUTO_RECALC=OFF */;

/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;

/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;

/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;

/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--

-- Current Database: `mysql`

--

CREATE DATABASE /*!32312 IF NOT EXISTS*/ `mysql` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci */ /*!80016 DEFAULT ENCRYPTION='N' */;

USE `mysql`;

...
```

可以看到，文件 backup.sql 本质就是一个文本文件，里面记录的就是一条条 SQL 语句，而这就是我们说的逻辑备份。要恢复逻辑备份非常简单，就是执行文件中的 SQL 语句，这时可以使用下面的 SQL：

```
mysql < backup.sql
```

虽然 mysqldump 简单易用，但因为它备份是单线程进行的，所以速度会比较慢，于是 MySQL 推出了 mysqlpump 工具。

命令 mysqlpump 的使用几乎与 mysqldump 一模一样，唯一不同的是它可以设置备份的线程数，如：

```mysql
mysqlpump -A --single-transaction --default-parallelism=8 > backup.sql

Dump progress: 1/1 tables, 0/0 rows

Dump progress: 25/37 tables, 881632/42965650 rows

Dump progress: 25/37 tables, 1683132/42965650 rows

......
```

上面的命令显示了通过 mysqlpump 进行备份。参数 –default-parallelism 表示设置备份的并行线程数。此外，与 mysqldump 不同的是，mysqlpump 在备份过程中可以查看备份的进度。

**不过在真正的线上生产环境中，我并不推荐你使用 mysqlpump，** 因为当备份并发线程数超过 1 时，它不能构建一个一致性的备份。见 mysqlpump 的提示：

![avadar](C:\XingMinYu\java后端\NOTE\figures\10.jpg)

另外，mysqlpump 的备份多线程是基于多个表的并行备份，如果数据库中存在一个超级大表，那么对于这个表的备份依然还是单线程的。那么有没有一种基于记录级别的并行备份，且支持一致性的逻辑备份工具呢？

有的，那就是开源的 mydumper 工具，地址：https://github.com/maxbube/mydumper。mydumper 的强大之处在于：

1. 支持一致性的备份；
2. 可以根据表中的记录进行分片，从而进行多线程的备份；
3. 对于恢复操作，也可以是多线程的备份；
4. 可以指定单个表进行多线程的恢复。

我们可以看到，mydumper 几乎是一个完美的逻辑备份工具，是构建备份系统的首选工具。我提供给你一个简单的 mydumper 的使用方法：

```mysql
mydumper -o /bak -r 100000 --trx-consistency-only -t 8
```

上面的命令表示，将备份文件保存到目录 /bak 下，其中：

1. 参数 -r 表示每张表导出 100000 条记录后保存到一张表；
2. 参数 –trx-consistency-only 表示一致性备份；
3. 参数 -t 表示 8 个线程并行备份。

可以看到，即便对于一张大表，也可以以 8 个线程，按照每次 10000 条记录的方式进行备份，这样大大提升了备份的性能。

##### **物理备份**

当然，逻辑备份虽然好，但是它所需要的时间比较长，因为本质上逻辑备份就是进行 INSERT … SELECT … 的操作。

而物理备份直接备份数据库的物理表空间文件和重做日志，不用通过逻辑的 SELECT 取出数据。所以物理备份的速度，通常是比逻辑备份快的，恢复速度也比较快。

但它不如 mydumper 的是，物理备份只能恢复整个实例的数据，而不能按指定表进行恢复。MySQL 8.0 的物理备份工具可以选择官方的 Clone Plugin。

Clone Plugin 是 MySQL 8.0.17 版本推出的物理备份工具插件，在安装完插件后，就可以对MySQL 进行物理备份了。而我们要使用 Clone Plugin 就要先安装 Clone Plugin 插件，推荐在配置文件中进行如下设置：

```mysql
[mysqld]

plugin-load-add=mysql_clone.so

clone=FORCE_PLUS_PERMANENT
```

这时进行物理备份可以通过如下命令：

```mysql
mysql> CLONE LOCAL DATA DIRECTORY = '/path/to/clone_dir';
```

可以看到，在 mysql 命令行下输入 clone 命令，就可以进行本地实例的 MySQL 物理备份了。

Clone Plugin 插件强大之处还在于其可以进行远程的物理备份，命令如下所示：

```mysql
CLONE INSTANCE FROM 'user'@'host':port

IDENTIFIED BY 'password'

[DATA DIRECTORY [=] 'clone_dir']

[REQUIRE [NO] SSL];
```

从上面的命令我们可以看到，Clone Plugin 支持指定的用户名密码，备份远程的物理备份到当前服务器上，根据 Clone Plugin 可以非常容易地构建备份系统。

对于 MySQL 8.0 之前的版本，我们可以使用第三方开源工具 Xtrabackup，官方网址：https://github.com/percona/percona-xtrabackup。

不过，物理备份实现机制较逻辑备份复制很多，需要深入了解 MySQL 数据库内核的实现，**我强烈建议使用 MySQL 官方的物理备份工具，开源第三方物理备份工具只作为一些场景的辅助手段。**

#### 增量备份

前面我们学习的逻辑备份、物理备份都是全量备份，也就是对整个数据库进行备份。然而，数据库中的数据不断变化，我们不可能每时每分对数据库进行增量的备份。

所以，我们需要通过“全量备份 + 增量备份”的方式，构建完整的备份策略。**增量备份就是对日志文件进行备份，在 MySQL 数据库中就是二进制日志文件。**

因为二进制日志保存了对数据库所有变更的修改，所以“全量备份 + 增量备份”，就可以实现基于时间点的恢复（point in time recovery），也就是“通过全量 + 增量备份”可以恢复到任意时间点。

全量备份时会记录这个备份对应的时间点位，一般是某个 GTID 位置，增量备份可以在这个点位后重放日志，这样就能实现基于时间点的恢复。

如果二进制日志存在一些删库的操作，可以跳过这些点，然后接着重放后续二进制日志，这样就能对极端删库场景进行灾难恢复了。

想要准实时地增量备份 MySQL 的二进制日志，我们可以使用下面的命令：

```
mysqlbinlog --read-from-remote-server --host=host_name --raw --stop-never binlog.000001
```

可以看到，增量备份就是使用之前了解的 mysqlbinlog，但这次额外加上了参数 –read-from-remote-server，表示可以从远程某个 MySQL 上拉取二进制日志，这个远程 MySQL 就是由参数 –host 指定。

参数 –raw 表示根据二进制的方式进行拉取，参数 –stop-never 表示永远不要停止，即一直拉取一直保存，参数 binlog.000001 表示从这个文件开始拉取。

MySQL 增量备份的本质是通过 mysqlbinlog 模拟一个 slave 从服务器，然后主服务器不断将二进制日志推送给从服务器，利用之前介绍的复制技术，实现数据库的增量备份。

增量备份的恢复，就是通过 mysqlbinlog 解析二进制日志，然后进行恢复，如：

```
mysqlbinlog binlog.000001 binlog.000002 | mysql -u root -p
```

### 7.2 备份策略

在掌握全量备份、增量备份的知识点后，我们就能构建自己的备份策略了。

首先，我们要设置全量备份的频率，因为全量备份比较大，所以建议设置 1 周 1 次全量备份，实时增量备份的频率。这样最坏的情况就是要恢复 7 天前的一个全备，然后通过 7 天的增量备份恢复。

对于备份文件，也需要进行备份。我们不能认为备份文件的存储介质不会损坏。所以，至少在 2 个机房的不同存储服务器上存储备份文件，即备份文件至少需要 2 个副本。至于备份文件的保存期限，取决于每个公司自己的要求（比如有的公司要求永久保存，有的公司要求保留至少近 3 个月的备份文件）。

所有的这些备份策略，都需要自己的备份系统进行调度，这个并没有什么特别好的开源项目，需要根据自己的业务需求，定制开发。

### 7.3 备份文件的检查

在我的眼中，备份系统非常关键，并不亚于线上的高可用系统。

在 18 讲中，我们讲到线上主从复制的高可用架构，还需要进行主从之间的数据核对，用来确保数据是真实一致的。

同样，对于备份文件，也需要进行校验，才能确保备份文件的正确的，当真的发生灾难时，可通过备份文件进行恢复。**因此，备份系统还需要一个备份文件的校验功能**。

备份文件校验的大致逻辑是恢复全部文件，接着通过增量备份进行恢复，然后将恢复的 MySQL实例连上线上的 MySQL 服务器作为从服务器，然后再次进行数据核对。

牢记，只有当核对是 OK 的，才能证明你的备份文件是安全的。所以备份文件同样要检查。

### 总结

今天我们学习了构建 MySQL 的备份策略，首先讲了 MySQL 数据库的全量备份逻辑备份、物理备份。接着学习了通过 mysqlbinlog 进行增量备份。通过全量备份和增量备份就能构建一个完整的备份策略。最后明确了要对备份文件进行检查，以此确保备份文件的安全性。

希望在学完这一讲后，你能将所学内容应用到你的生产环境，构建一个稳定、可靠，即便发生删库跑路的灾难事件，也能对数据进行恢复的备份系统。
