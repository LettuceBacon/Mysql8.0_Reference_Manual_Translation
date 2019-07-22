## 15.3 InnoDB多版本并发控制

> 原文地址：https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html

InnoDB是一个多版本存储引擎：它保存改变的行的旧版信息，以此来支持如并发和回滚的事务特性。这些信息用一种称之为`rollback segment`的数据结构保存在表空间里(类似Oracle里的一种数据结构)。InnoDB同时使用这些信息来保存较早的行数据用于一致性读(`consistent read`)。

对于内部实现，InnoDB为每一行数据增加三个列。一个6字节的`DB_TRX_ID`字段存储最近插入或修改的事务标识。同时，删除操作在内部也被当做更新，并用一个特殊的位来标记它已被删除。每一行同时包含一个称之为回滚指针的7字节`DB_ROLL_PTR`字段。回滚指针指向一个写到回滚数据段的undo log日志。如果某一行数据被更新，undo log日志会记录用以恢复原始数据的必要信息。一个6字节的`DB_ROW_ID`字段会记录每当新行插入时会递增的行号。如果InnoDB自动创建聚簇索引，那么这个索引会包含这个行号值。否则，`DB_ROW_ID`列不会出现在任何索引中。

存储在回滚数据段(rollback segment)的undo logs日志被区分为插入和更新两种日志。insert undo logs仅在事务被回滚时用到，一旦事务提交就可以立即删除。update undo logs除此之外还被用于一致性读，只有当事务在一致性读中分配的所有快照都不需要update undo logs里的信息来恢复数据行早期的数据时才能被删除。

定期提交你的事务，包括这些仅仅在一致性读中发起的事务。否则，InnoDB不能及时的从update undo logs删除数据，导致回滚数据段越来越大，最终塞满你的表空间。


一个存在回滚段里的undo log所需的物理空间通常小于相应的插入或更新的行数据。你可以用这个信息来计算你的回滚段所需要的空间。

在InnoDB的多版本体系中，当你用一个SQL语句删除一行时，数据并不会立马被物理删除。只有当删除行对应的update undo log被删除时，相应的行和它的索引才会被物理删除。这种删除操作被称为清除(`purge`)，这种操作的速度非常快，通常能和SQL删除语句的先后顺序保持一致。

如果你向一张表中以大约相同的比率添加和删除小批次的行时，清除线程可能会滞后并导致表由于没用的行数据而越来越大，最终由于磁盘限制导致所有的操作都变的非常慢。在这种情况下，限制新行的操作，并通过调整`innodb_max_resource_lag`系统变量来分配更多的资源给清除线程来解决这个问题。更多详情请参阅`第15.13节 InnoDB启动选项和系统变量`。

### 多版本和二级索引
InnoDB的多版本并发控制(MVCC)对待非主键索引和聚簇索引是不一样的。聚簇索引中的记录就地更新，并且它们的隐藏列指向可以重建早起记录的undo log实体。和聚簇索引不同的是，非主键索引不包含隐藏的列，也不就地更新。

如果一个主键建索引的列被更新，旧的二级索引记录会被标记为删除，新的记录会被插入，并且最终被标记为删除的记录会被清除线程清除。当一个二级索引记录被标记为删除或者二级索引页被一个更新的事物删除时，InnoDB去聚簇索引中寻找数据。在聚簇索引中，会检查记录的`DB_TRX_ID`，当开启了一个读事物后如果记录被修改，InnoDB会从undo log中拿到正确版本的数据。

如果一个二级索引记录被标记为删除或者二级索引记录页数据被一个更新的事物更新，覆盖索引技术将不可用。取而代之的是InnoDB从聚集索引中查找记录。

然而，如果ICP优化被启用，并且WHERE查询中的部分判断能够仅仅使用索引的字段，MySQL数据库仍旧会把这部分WHERE判断下沉到存储引擎层。如果没有找到匹配的行，聚集索引的查找就可以避免。如果查找到匹配的行，即使是在标记为删除的记录中，InnoDB会从聚簇索引中寻找记录。