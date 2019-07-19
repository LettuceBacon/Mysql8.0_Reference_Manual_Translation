## 15.1.1 InnoDB表的优势

你将会发现使用InnoDB会有如下的优势：

- 如果你的服务器由于软硬件故障到导致宕机，不管此时你的数据库在处理什么，重启数据库后你都不需要做任何额外的工作。InnoDB的故障恢复功能能够自动把故障发生之前提交的任何修改持久化，并撤销任何未提交的修改。仅仅重启数据库然后什么都不用做。

- InnoDB存储引擎能在数据被访问的时候用自己的buffer pool来把表和索引数据缓存在内存当中。访问频率高的数据直接从内存读取。这种缓存机制能够应用在多种数据场景下来加速它们的处理。对于单独部署的数据库服务器，通常高达80%的物理内存会被分配给buffer pool。

- 如果你把相关联的数据分开存储到不同的表中，你可以使用外键来强制关联完整性。当更新或删除数据时，关联的数据会自动更新和删除。试图向关联表中插入一条不在主表里的数据，脏数据会被自动剔除。

- 如果磁盘或内存中的数据遭到破坏，在你使用这些数据前，校验和机制会向你发出报警。

- 当你在设计数据库时为每张表都选择了一个合适的主键列时，对这些主键列进行的操作都会被自动优化。对主键进行WHERE查询，ORDER BY排序，GROUP BY和JOIN操作时都是非常高效的。

- 增删改操作通过一种称之为change buffering的机制来自动进行优化。InnoDB不仅允许对同一张表并发的进行读写，同时缓存数据让磁盘IO变为顺序读写（译注：对磁盘顺序读写的性能远远高于随机读写）。