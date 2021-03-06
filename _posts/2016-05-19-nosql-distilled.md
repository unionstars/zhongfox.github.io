---
layout: post
tags : [缓存, redis, nosql, 分布式, 读书笔记]
title: 《NoSql 精粹》读书笔记

---

## 为什么使用NoSql

* RMDB价值

  * 数据持久化
  * 并发: RMDB 使用事务控制并发; 事务执行失败可以回滚
  * 集成: RMDB可以做为"共享数据集成"提供给多个应用使用, 达到数据共享; 同时RMDB有并发控制机制.
  * 标准模型: 各种RMDB大同小异

* RMDB 阻抗失衡

  RMDB中的表(关系), 行(键值对的元组)无法直接表示实际内存中的数据结构, 比如嵌套记录, 列表等

  阻抗失衡要求关系和实际数据结构之间必须转译

  应对:

  * 面向对象数据库: 消失鸟
  * ORM: 轻松解决阻抗失衡, 但是"框架本身就成了问题", 查询性能下降
  * NoSql

* 数据库在应用共享上的划分

  * 集成数据库: 多应用同时读写一个数据库, 应用之间数据完整性难以保证, 完整性需要数据库来负责(RMDB有固有schema在这里是一个优势)

    这也是RMDM战胜面向对象数据库的一个原因

  * 应用程序数据库: 指定的数据库由一个应用维护, 该应用代码维护数据完整性, 数据结构对其他数据库不透明

    通过接口对外提供服务, 这是面向服务架构的特征之一, 交互数据结构更灵活, 比如使用json/xml/二进制协议

    内部数据与外部通信解耦, 数据库技术选择余地更大

* RMDB 与 集群

  RMDB 分片实现水平扩展, 但是应用程序必须控制分片, 查询, 完整性, 事务, 一致性等跨分片难以实现

  RMDB 按照单机计费

* NoSql

  * 不使用关系模型
  * 专注大数据, 集群
  * 无模式: 提高开发效率, 优化阻抗失衡
  * 适合"应用程序数据库", 不适合"集成数据库"
  * 开源
  * 影响: 混合持久化

---

## 聚合数据模型

* 数据模型: 数据库组织数据的方式

  RMDB使用关系数据模型, 每种NoSql实现各种不同的数据模型, 主要有:

  * 键值
  * 文档
  * 列族
  * 图

  其中, "键值" "文档" "列族" 都是面向聚合

* 聚合 affregate

  相关联对象作为一个整体单元来操作, 比如列表或者嵌套记录

  聚合更容易完成原子操作, 复制, 分片, 同时对应用程序更友好

  RMDM 没有聚合实体的概念, 是聚合无知的, NOSQL中图数据库也是聚合无知

  聚合规划应该更多的由**数据访问方式**决定

  聚合边界一般难以界定, 如果有多种差异比较大的访问形式, 面向聚合反而是比较困难

  因此, 如果没有一种主导的结构, 使用聚合无知可能更好

  面向聚合不支持跨聚合的ACID事务, 但是单个聚合上是可以进行原子操作的, 因此如果需要跨聚合原子操作, 需要应用程序实现, 但是使用聚合应该将大部分原子操作局限到单聚合内部

* 键值聚合: 聚合不透明, 可以存储任何数据, 基本通过key查询

* 文档聚合: 存放数据有限制, 有结构, 可以局部获取, 可以根据聚合内容创建索引, 可以根据文档内部信息进行查询

* 列聚合: TODO

---

## 数据模型详解

* 关系:

  数据模型间不可避免存在关联, 各种NoSql也以不同方式支持关系

  面向聚合的数据库不适合处理大量关系, 这种情况更应该使用关系型数据库(SQL 提供 JOIN, 但是效率会随着关系的复杂度而降低)

* 图数据库

  适合处理: 关系复杂, 数据模型简单

  数据结构: 节点(node), 边(edge)

  与RMDB比较:

  * RMDB JOIN 效率低
  * 图数据库有专门为图关系设计的查询操作, 关系遍历快: 因为在插入时花时间构建关系数据, 提满足高效的关系查询 (权衡以提供满足需求)

  与面向聚合数据库比较:

  * 图数据库重视关系
  * 图数据库通常运行在单机, 不是分布式, 支持ACID事务
  * 两者都不使用(标准)关系模型

* NoSql 共同的特征: 没有 Schema

  优势:

  * 灵活自由
  * 处理格式不一致的数据, 避免出现大量null (RMDB)

  劣势:

  * 应用程序需要实现理解NoSql数据结构的"隐含模式" (对数据结构进行假设)
  * 不适合多程序操作, 可作为应用程序数据库, 对外提供web服务实现共享

  大多数情况下, 如果发现存储数据类型不统一, 优选无模式数据库

* 物化视图 materializd view

  * 面向聚合不擅长动态处理聚合内部数据的关联, RMDB在这方面有优势
  * RMDB还提供了视图, 用于展示不同的数据关系结构
  * 视图有开销, 因此出现"物化视图": 预处理, 非实时
  * NoSql使用map-reduce 来实现物化视图

---

## 分布式模型

* 复制和分片是正交的:

  * 复制: 同一份数据拷贝到多个节点; 有两种: 主从式, 对等式
  * 分片: 不同数据分布到不同节点

* 单一服务器

  在不需要分布数据时, 总应该选择单一服务器

* 分片

  为了充分水平扩展(将负载能力提升为 `单机能力*N`), 数据分布考虑:

  * 使用聚合, 把经常一起用的数据聚合到一起
  * 将指定分片节点部署到离指定节点近的位置(物理位置)
  * 保证各分片机器负载平均 (保持负载均衡)

  部分NoSql提供了自动分片功能: 数据库决定具体数据分布, 并引导应用访问适合的分片

  分片可以同时提升读和写的效率

  复制可以改善读的能力(当然还有高可用)

  分片不能改善"故障恢复能力", 这方面的改善只是: 故障发生会影响分片部分用户

* 主从复制 master-slave

  * 水平扩展读能力, 稍微缓解主机写能力
  * 增强读取的故障恢复能力(需要确保应用实现读写分离)
  * 如果支持主节点失效转移, 可以实现即时备份 (可以理解为有即使备份的单一服务器)
  * 缺陷: 数据不一致, 即时是"即使备份"也存在数据不一致(失效转移期间的写丢失)

* 对等复制 peer-to-peer

  * 没有主节点概念, 所有节点对等, 存储所有数据, 都可以写
  * 对写入有故障恢复能力
  * 实现复杂, 需要在节点间协调同步写入数据, 需要解决写入冲突
  * 一致性问题

* 复制和分片结合

  * 主从复制+分片: redis cluster
  * 对等复制+分片: 列族?

---

## 一致性

### 一致性问题分类:

* 更新一致性:

  * 顺序一致性:

    并发写入, 顺序处理, 后者写入覆盖前者写入, 前者写入丢失. 发生数据不一致

    解决:

    * 乐观锁: 条件更新 compare and set
    * 悲观锁: 加锁再更新

  * 复制一致性

    对等节点同时写入不同数据, 造成写冲突

    解决:

    * 自动合并
    * 呈交用户处理

    另一种不一致: 主从复制, 当出现脑裂, 主区可写, 从区不可写, 出现更新不一致

* 读取一致性

  * 逻辑一致性

    逻辑上应该原子性处理的多个数据, 写之间存在间隔, 间隔之间的读取不一致

    间隔叫做"不一致窗口"

    解决:

    * 使用事务
    * 面向聚合的不支持事务, 可以将逻辑关联的数据封装为单个聚合

  * 复制一致性

    复制数据传播过程中, 2个节点看到的数据不一致

    缓存也是一种不满足复制一致性的场景

    解决:

    * 最终一致性

### 关于最终一致性:

* 有的情况可以容忍较长时间的不一致窗口

* 会话一致性是最终一致性的一种, 实现:
  * 黏性会话
  * 版本戳

### 放宽一致性约束:

* 事务可以提供较强一致性, 但事务对系统性能影响较大

* CAP:

  单机是一个无P的CA系统, 无法满足分区耐受

  也有满足CA的分布式系统, 也就是如果就node 挂了, 系统就挂了

  因此CAP理论的意义在于必须支持P, 在CA间权衡

* 权衡: 通常情况是尽量保证可用性, 适当降低一致性, 保证最终一致性

  * 写入不一致和可用性: 保证可用性, 运行写入不一致, 然后在结算时合并或让用户确认, 需要求助领域专家(结合业务), 比如购物车

  * 读取一致性和可用性: 需要知道用户对陈旧数据的容忍程度, 需要知道不一致窗口的长度, 结合需求对复制配置参数进行配置

* BaSE: 很做作, 恩, 我也觉得

* 一致性和可用性的权衡, 不如说是一致性和延迟之间的权衡

  参与处理数据的节点越多, 一致性越好, 但节点的增加, 延迟增大, 可以认为可用性在降低

### 放宽持久性约束

* 持久性权衡: 延迟和持久性的权衡, 数据重要程度的权衡, 持久性一定程度代表一致性.

  需要结合业务, 对于会话等临时信息, 可重建, 不重要信息可以放到内存, 降低操作延迟

* 复制持久性:

  master-slave复制中, master的写入在同步到slav之前, 发送了失效转移, 部分写入数据丢失

  可以配置同步完成后才告知用户写入成功, 这会增大延迟

  需要结合数据重要程度

### 权衡实现: 仲裁


* 名词

  * 复制因子N: 副本个数, 不等于节点个数. 增大影响: 故障恢复能力增加, 保持一致性更难更慢
  * 写入确认节点数W: 增大影响: 写入一致性加强, 写入速度变慢, (应该可以使得读取一致性实现更快)

* 对等分布环境强一致性保证:

  * 写仲裁: 确保不出现写冲突 `W>N/2`

  * 读仲裁: 确保读到最新数据 `R+W>N`

* 主从一致性保证: 读写都针对master

* 通常, N为3可以获得较好的故障恢复能力

---

## 版本戳 Version Stamp

* 事务不是万能的, 事务需要额外人工干预, 事务无法封装所有操作, 但是这些都是可以通过版本戳完成

* 离线并发

      用户A将数据R（C1,C2）读取到A的浏览器中。
      用户B将数据R（C1,C2）读取到B的浏览器中。
      用户A在浏览器上将数据修改为R（C1’，C2），同时更新到数据库。
      用户B在浏览器上将数据修改为R（C1，C2’），同时更新到数据库。
      上述过程存在两个问题，第一，第4步B在修改数据的时候数据库中的数据和B的浏览器中数据已经不一致了；第二，如果程序按照哪个字段变化在数据库中更新哪 个字段的方式处理的话，那么经过上述四步修改，数据库中R行的内容是（C1’,C2’），这和A或者B的想法都不同（A认为是（C1’，C2），B认为是 （C1，C2’））。上述过程中A对数据库的修改过程或者B对数据库的修改过程，都是无法根据数据库的最新内容做修改，所以称为为离线。A和B同时对记录R进行修改叫离线。以上的环境叫离线并发

* NoSql 中**乐观离线锁**: 条件更新的一种形式, 操作前对比存储的版本戳

  类比HTTP中的etag

  处理其中也有类似机制, CAS

* 版本戳的几种实现:

  * 计数器:

    优点: 短小, 可以比对新旧; 缺点: 需要主节点来统一生成;

  * GUID:

    优点: 任何client/server都可以生成; 缺点: 太长, 且不能比对新旧;

  * 内容hash:

    优缺同GUID

  * 更新时间戳:

    优点: 短小, 可以比对新旧, 任何节点都可以生成; 缺点: 节点需要时钟同步, 精度过低可能冲突

  实际中可以将几种进行结合使用

* 多节点下版本戳

  数组式版本戳(TODO理解不够深入)

---

## Map-Reduce

面向聚合的兴起得益于集群的增长, 集群不仅改变了数据存储规则, 而且改变了数据计算规则. 如果大了数据存储在集群, 需要有新的思路来处理数据计算流程.

map-reduce 可以把运算任务分布到多台计算机节点中, 同时网络数据传输量不大

* map: 一个函数, 分布到节点中并发执行 输入是一个个聚合, 输出若干键值对, 把关键字下的数据汇集为集合, 交给reduce

* reduce: 一个函数, 操作统一关键字的数据, 输入多个同关键字的数据. 一定是相同关键字, 一次多个.

TODO

---

## 模式迁移

### RMDB 模式迁移

这小节意义不大

* 编写数据库变更脚本 (Rails中db/migrate/)

  * 体现版本, 维护变更顺序: 脚本文件名数字递增, 版本数字入库
  * 版本数字可以用于寻找相关脚本, 执行未执行的迁移脚本
  * 脚本既包括模式修改, 也包括所需数据迁移操作

* 使用数据库diff技术(??)

* 迁移已有项目 (之前没有维护migrat脚本的项目?)

  * 提取现有数据库结构到脚本, 后续同上
  * 对公用RMDB, 需要注意先后兼容(???)

### NoSql 模式迁移

* 无模式的Nosql中, 数据的模式需要应用程序来维护

* 无模式数据库迁移也可以借鉴强模式数据库迁移技术

* 增量迁移:

  (存储)模式变更后, 应用代码修改为兼容新老模式, 然后再次写入时按照新模式写入

  应用程序中可能存在多种对象版本; 增大对象设计复杂度

  需要注意尽量缩短过渡期

  数据库中可以存在一个`schema_version`存储模式版本, 协助完成增量迁移

  在领域和数据库之间一定要有适当的转译层, 用于把模式变迁的代码局限在转译层而不是整个应用程序

* 图数据库迁移 TODO

## 混合持久化

* 不同的需求对存储的需求不一致, 不同的数据库擅长处理的数据结构和数据量也不一样, 需要分场景使用

  不同的数据库设计目标不同, 并非所有问题都能用一种数据库优雅解决

  混合持久化是使用不同数据库解决一个或多个应用的存储需求

* 将直接数据库操作封装为服务, 可以在服务内部完善数据库(使用混合策略)

## 超越NoSql

TODO

## 选择合适的数据库

TODO
