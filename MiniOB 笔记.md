# MiniOB 笔记

# MiniOB 概述

## 数据库架构

1. 存储
2. 事务
3. SQL

**存储：**  相当于关系数据库的数据结构

**事务：**  相当于关系数据库的算法

**SQL：**  相当于关系数据库操作的描述语言，其实就是相当于接口

‍

### 存储

哈希表、B+树

‍

### 事务

* 原子性 （Atomicity）
* 一致性 （Consistency）
* 隔离性 （Isolation）
* 持久性 （Duration）

简单来说，事务要么成功，要么失败，事务之间互不影响，且事务完成过后对数据的影响是永久的。

‍

### SQL

我理解为：操作数据库的语言接口

‍

## MiniOB 框架

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/network-asset-image-20240929231655-bhe880v-20240930095403-2kuopew.png)​

简单来说，分为**客户端**、**服务端**。

## SQL处理流程

```shell
# 解析 SQL
网络模块 -> 解析模块 -> (语法树)

# 转换对象，预检
PlanCache -> Resolver -> (真实对象)

# 这两个部分算作一个优化模块，可能会循环优化
Transformer -> 优化 -> (查询计划)

# 执行
执行模块 -> (结果) -> 返回

# Executor 会访问底层模块
```

‍

---

> 表格来自：MiniOB文档

|步骤|说明|
| --------------------------------| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Parser（词法/语法解析模块）|在收到用户发送的 SQL 请求串后，Parser 会做词法/语法分析，将字符串分成一个个的“单词”，并根据预先设定好的语法规则解析整个请求，将 SQL 请求字符串转换成带有语法结构信息的内存数据结构，称为语法树（Syntax Tree）。|
|Query result cache|直接对 SQL 进行硬解析，若发现命中，将直接绕过下面所有过程，返回结果给应用方。|
|Resolver（语义解析模块）|Resolver 将生成的语法树转换为带有数据库语义信息的内部数据结构。在这一过程中，Resolver 将根据数据库元信息将 SQL 请求中的 Token 翻译成对应的对象（例如库、表、列、索引等），生成的数据结构叫做 Statement Tree。|
|Plan Cache（执行计划缓存模块）|执行计划缓存模块会将该 SQL 第一次生成的执行计划缓存在内存中，后续的执行可以反复执行这个计划，避免了重复查询优化的过程。  Resolver 中生成的 Statement Tree 将会与该模块中的执行计划做匹配，若匹配命中，Plan Cache  直接将其物理执行计划发送到 Executor 中进行执行。|
|Transfomer（逻辑改写模块）|分析用户 SQL 的语义，并根据内部的规则或代价模型，将用户 SQL  改写为与之等价的其他形式，并将其提供给后续的优化器做进一步的优化。Transformer 的工作方式是在原 Statement Tree  上做等价变换，变换的结果仍然是一棵 Statement Tree。|
|Optimizer（优化器）|优化器是整个 SQL 请求优化的核心，其作用是为 SQL 请求生成最佳的执行计划。在优化过程中，优化器需要综合考虑 SQL  请求的语义、对象数据特征、对象物理分布等多方面因素，解决访问路径选择、联接顺序选择、联接算法选择、分布式计划生成等多个核心问题，最终选择一个对应该 SQL 的最佳执行计划。|
|Code Generator（代码生成器）|将多个算子合并到一起，生成一个更高效的算子。|
|Executor（执行器）|启动 SQL 的执行过程。* 对于本地执行计划，Executor 会简单的从执行计划的顶端的算子开始调用，根据算子自身的逻辑完成整个执行的过程，并返回执行结果。<br /><br />* 对于远程或分布式计划，将执行树分成多个可以调度的子计划，并通过 RPC 将其发送给相关的节点去执行。|

---

‍

‍

## MiniOB 线程模型

### SEDA

使用 `SEDA`​ 来实现。

这里在第一阶段，我没有去深入了解。

简单来说，就是并发执行，由线程池消费用户请求。

### 事件的生命周期

```shell
创建事件 -> 进入线程池事件队列 -> 被调度执行 -> 事件生命周期结束
```

### 事件的执行过程

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/network-asset-image-20240929234426-fxca13a-20240930095411-6ai8bi7.png)​

‍

### 代码解读

~~国庆在看吧，细看的话太多了~~

# 存储器的层次结构

简单来说，就是从**快**到**慢**，从**易失**到**非易失**。

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240929234858-2o5mc14.png)​

‍

## 虚拟存储器

~~操作系统上过的，虽然忘记了~~

假设是 `32`​ 位操作系统，那么每个进程最大的内存大小为 `4G`​，但是操作系统不可能为每个程序都分这么多运行内存。这种情况下，使用**虚拟地址空间**来解决这个问题。

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/network-asset-image-20240929235221-guouuii-20240930095430-mw9j55v.png)​

由于用到了**磁盘**，加上内存页的调度可能会涉及到CPU从磁盘中读取数据，但磁盘到内存这一个过程需要花费大量时间（相对）。

那么，在大并发的情况下，如果经常性缺页，就会导致SQL执行返回结果的过程很慢，还会占用系统大量资源。

**总结：** 数据库要减少缺页，体验才好

‍

‍

## 磁盘存储器

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/network-asset-image-20240929235711-a29uxw2-20240930095449-64xlyza.png)​

就是**机械硬盘**。

**属性：**

* 盘面数量
* 磁道数量
* 扇区数量
* 扇区大小

### 磁盘空间计算

$$
磁盘空间 = 盘面数量 × 磁道数量 × 扇区数量 × 扇区大小
$$

**一个例子：**

* 8 个盘片，16 个盘面。
* 每个盘面有 2\^16\=65536 个磁道。
* 每个磁道 256 个扇区。
* 每个扇区 4096 个字节。

$磁盘空间 = 16×65536×256×4096Bytes = 1TB$

‍

### 磁盘读取性能

* **寻道时延**  
  磁臂移动到指定磁道所需要的时间。
* **旋转时延**  
  把扇区移动到磁头下面的时间。
* **传输时延**  
  从磁盘读出或将数据写入磁盘的时间。

$总延时 = 寻道时间 + 旋转时间 + 传输时间$  

‍

### 磁盘故障

**故障类别**

* 间断性故障
* 介质损坏
* 写故障
* 磁盘崩溃

一般是磁盘本身问题。

‍

**检测机制**

* 校验和技术

‍

**防范方法**

* 磁盘阵列

加钱买硬盘......

‍

## 块与记录

* 记录

  按照若干个字段元素组成的数据集合

* 数据块

  由多个数据记录元素组成的一个数据集合，在物理上对应的是一块连续的存储空间

**块**包含多个**记录**

**记录**包含多个**字段**

‍

### 记录类型

主要就两类：**变长**、**定长**。

---

变长： `varchar`​、`blob`​、`text`​ 等等

定长：`int`​、`date`​等等

---

### 记录的存储结构

>  **总结：** 要对齐。

**对齐**会导致空间占用比**不对齐**多一点，但是提高了数据读取性能。

### 块的存储结构

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket//network-asset-image-20240930001302-59da78s-20240930095451-0ex6jr9.png)​

**就是：**  **​`Header`​**​  **+**  **​`Data`​**​

> **Header：偏移量、修改时间戳、块的存储位置信息**
>
> 方便**块**自身的维护和管理

> **Data：数据存放位置**

‍

## 变长数据和记录

### 定义

* 数据大小不确定
* 重复字段
* 格式不确定
* 大字段

比如说姓名，有三个字的，也可能有十几个字的，不定长；全国的姓名会有重名，但是每个名字重复次数不一样。

​`JSON`​ 这种形式的类型，里面具体长啥样不确定。

大字段，可以说是原始数据，这个就更不确定了。

### 类型

* 变长
* 重复
* 可变
* 大值
* 二进制

### 变长字段的记录

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930002047-tws6icv.png)​

元信息 + 数据块

### 重复字段的记录

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930002052-wsh8nlf.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930002142-olni6lr.png)​

将**重复字段**放到块的最后，统一存储，方便查询。

### 可变格式的记录

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930002145-oicxylv.png)​

假设要记录一个人的信息，包括姓名、投资的餐馆等，可能字段固定但是内容不固定。这个时候可以使用标记（如姓名）以及类型（如  string）加上长度来记录，例如图中的 N 表示 name，S 表示 string 类型，14 表示 name  的长度。这样做的好处是即便不知道存储的格式，但是可以把一些特征标识保存起来，最后根据标识解析出数据。

**编码+类型+长度+数据**

 **（N，S，14，数据）**

 **（R，S，16，数据）**

### 大值类型的记录

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930002207-obysefs.png)​

如图记录 2 存储时，分成了两部分进行存储，分别是记录 2-a 和记录 2-b，其中在存储记录 2-a  的时候，需要额外的存储空间来保存一些数据：一是标识，表明这是一个记录的片段；二是片段的序号，如这是第 1 个片段，或者第 N  个片段；最后还要保存指向下一个片段的指针，用于跨块之间的访问。

**有点类似于链表的形式**

### 二进制大对象

连续的块，块的链表

### 行存与列存

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930002219-016gb37.png)​

**列存**

列存存储的数据类型一般都一样，方便进行压缩存储，压缩比高；但是读写性能比行存稍差。

## 记录的修改

一般是**增**、**删**、**改**。

### 插入

```shell
找到空闲块 -> 记录插入空闲块 -> 维护元信息
```

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket//image-20240930003249-8bp779u.png)​

**首部：**  元信息，偏移信息

找到空闲位置，插入一个记录，记录偏移信息（写入偏移表）

### 删除

数据库中一般是在**块**的前面打一个标记，等数据库空闲之后在进行删除回收空间。

### 修改

**定长修改**

数据可以直接覆盖。

**变长修改**

```shell
找空闲位置 -> 写入新数据 -> 旧的数据打删除标记

元数据中维护新数据的偏移信息
```

## MiniOB 存储实现原理

这个是代码了，二期在看。

‍

# 索引结构

前两节回顾了数据结构的内容。

## B+树

## 散列表

## LSM

* [ ] 这一部分还需要深入查看

>  **全称：日志结构合并树**
>
>  **核心思想：** 将内存中的缓存数据以记录日志的形式追加写入到存储介质

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930004408-h76chn6.png)​

---

一个写入操作，首先写入 C<sub>0 。</sub>

C<sub>0</sub> 的数据与 C<sub>1</sub>合并，结果存入 C<sub>1</sub>。

同理，按照层级往下面存储。

**以上的过程称为**​**​`Compaction操作`​**​

---

> LevelDB 数据库对 LSM-Tree 的实现

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930004808-dafv71h.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005017-php2igx.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005017-php2igx.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005029-gvuirj6.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005034-tqbqqo1.png)​

>  `Memtable`​：内存表（读写）
>
>  `Innutable Memtable`​： 内存只读表
>
>  `SSTable`​：顺序字符串表（KV数据）

放了很多图片，上面的过程其实可以简略为：

```shell
# 内存 =======================================

内存读写表 -> (满) -> 内存只读表

# 内存 =======================================

# 外存 =======================================

L0 -> (满) -> 合并操作 -> L1
L1 -> (满) -> 合并操作 -> L2
以此类推.....

# 外存 =======================================
```

其中合并操作可能存在数据重叠，就需要对合并后重复数据进行删除操作。

‍

## MiniOB B+树 实现

‍

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005530-wh8lkaq.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005537-c96kgjg.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005541-uuc207l.png)​

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005545-duwdx8t.png)​

和数据结构中的**B+树**差不多

### 插入

和 B+Tree 差不多

### 修改

和 B+Tree 差不多

### 删除

和 B+Tree 差不多

‍

# SQL 引擎

这里就是介绍那些模块的细节了

---

**左：MySQL**

**右：OceanBase**

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930005759-kza0mvy.png)​

‍

## SQL 执行顺序

```sql
SELECT <字段名>
FROM <表名>
JOIN <表名>
ON <连接条件>
WHERE <筛选条件>
GROUP BY <字段名>
HAVING <筛选条件>
ORDER BY <字段名>
LIMIT <限制行数>

# 此处不包含 subquery
```

>  执行顺序

```sql
FROM -> ON -> JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT
```

## Parser

SQL 解析成语法树。

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930010253-xzk84au.png)​

## Resolver

语法树进行进一步检查。

* 关系或属性的检查
* 类型的检查
* 类型推演
* 虚视图的展开

## Transformer 和 Optimizer

> 重点：火山模型

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930010544-i4nbip0.png)​

## Executor

### 火山模型

在火山模型中，所有的代数运算符（operator）都被看成是一个迭代器，它们都提供一组简单的接口：open()—next()—close()，查询计划树由一个个这样的关系运算符组成，每一次的 next() 调用，运算符就返回一行（Row），每一个运算符的 next() 都有自己的流控逻辑，数据通过运算符自上而下的 next()  嵌套调用而被动的进行拉取。

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930010658-bt0dhhg.png)​

**优点：运行只需要少量内存。**

**缺点：大数据量下，CPU消耗高。**

## Fast-Parser 和 Plan Cache

如果发起相同的请求，就跳过SQL解析，直接生成执行计划。

**Fast-Parser：对SQL进行常量替换，方便做 match**

## 基础代数符号与操作

就是那些**并**、**交**、**差**、**选择**、**投影**、**连接**。

还有一个**笛卡尔积**的概念。

### 聚合

接触到比较多的是聚合函数（count、min、max、sum、avg等等）。

### 物化和流水线计算

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930011424-k5sm1qe.png)​

我的理解：每一次迭代，都存有一部分数据，下一次能够直接取到上一次的数据。代价是内存消耗较大，性价比不高。

## 关系表达式的等价规则转换

* [ ] 数据库概论里面学的转换，待补充

## 执行计划的选择

* [ ] 待补充

## 代价

扫描底层数据（涉及到IO）的代价，表连接时，需要临时空间（笛卡尔积）的代价。

# 事务引擎

## 事务简介

>  事务（ACID）

引用：事务

‍

## 故障类型

* **数据输入错误**

  用户粗心大意输错了几个字符
* **介质故障**

  磁盘部分区域坏了，或者整个损坏
* **系统故障**

  突然断电，内存里面的东西没有写入到磁盘（sync）

* **机房故障**

  比如阿里云新加坡这几天的火灾

## 事务模型

### 事务模型

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930012557-1qqyz2r.png)​

* **事务管理器**统筹管理事务的执行。
* **查询处理器**负责解析 SQL 命令。
* **缓冲区管理器**负责维护内存缓冲区和刷写数据。
* **日志管理器**负责维护日志。
* **恢复管理器**负责在系统重启后恢复数据。

### 事务原语

>  我的理解：一组不可分割，不可中断的操作。

数据库中主要对应数据的转移操作，例如：

1. INPUT(X)：将数据库元素 X 从磁盘拷贝到缓冲区。
2. READ(X, t)：将数据库元素 X 从缓冲区拷贝到事务的局部变量 t。
3. WRITE(X, t)：将局部变量 t 的值拷贝到缓冲区的数据库元素 X，如果 X 不在缓冲区，先执行 INPUT(X)。
4. OUTPUT(X)：将数据库元素 X 从缓冲区拷贝到磁盘。

## 日志记录

日志：由多个日志记录组成。

```shell
[START T] 	事务开始
[COMMIT T] 	事务完成
[ABORT T] 	事务无法完成
[T, X, v] 	undo 日志
```

## undo 日志

> **回滚日志**

用于系统从故障中恢复。

### 恢复数据

遍历日志，扫描事务的执行情况，一步步恢复。

### 检查点机制

先保证前一部分的日志没问题，那么下次恢复的时候就不需要撤销这些操作。

### 动态检查点

前几个的缺点，恢复时无法执行新的事务。（系统看起来像宕机了）

* [ ] 有点麻烦，需要细看

## redo 日志

>  都是数据恢复

|区别|redo|undo|
| :----------: | :--------------------: | :--------------------: |
|未完成事务|撤销|忽略|
|已提交事务|忽略|重做|
|顺序|先写磁盘，在记录日志|先记录日志，再写磁盘|
|数据|记录旧数据|记录新数据|

## 隔离级别

* **脏读（dirty read）**

  指一个事务读取了另一个事务还未提交的修改。

* **不可重复度读（nonrepeatable read）**

  指在一个事务过程中，可能出现多次读取同一数据但得到的值不同的现象。
* **幻读（phantom read）**

  指在一个事务中，当查询了一组数据后，再次发起相同查询，却发现满足条件的数据被另一个提交的事务改变了。

‍

|隔离级别|脏读|不可重复读|幻读|
| ----------| ----------| ------------| ----------|
|读未提交|可能出现|可能出现|可能出现|
|读提交|不能|可能出现|可能出现|
|可重复读|不能|不能|可能出现|
|可有序化|不能|不能|不能|

## 锁

* [ ] []()TODO

## 时间戳

* [ ] TODO

## 多版本并发控制

### 版本控制协议

**基于时间戳**

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930014340-bf9yvmd.png)​

**基于两阶段加锁**

​![image](https://raw.githubusercontent.com/tecwds/picgo-repo/bucket/image-20240930014344-ut9hzub.png)​

### 快照隔离

* [ ] TODO

‍

# 内核基础

**性能**、**利用率**、**可诊断**

# 向量数据库

> 存储、检索、混合检索、高可用、分布式

‍
