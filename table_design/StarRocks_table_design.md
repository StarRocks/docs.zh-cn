# 理解 StarRocks 表设计

## 列式存储

![img](https://starrocks.feishu.cn/space/api/box/stream/download/asynccode/?code=YzQ1MjQwM2MyZTdiMWQ1YWUyMDUzZTgyYzY5NzlkZTRfVVVQS0dERmpOczV1cWxpUjQ0Tk9vcVkxd05PMDFCYVdfVG9rZW46Ym94Y25XZzc2QjNWU0U2NFl0eEIxOWYyUjhiXzE2NTY0NjkxNDc6MTY1NjQ3Mjc0N19WNA)

StarRocks 中的表和关系型数据相同，由行和列构成。每行数据对应用户一条记录，每列数据具有相同的数据类型。所有数据行的列数相同，可以动态增删列。在 StarRocks 中，一张表的列可以分为维度列（也称为 Key 列）和指标列（也称为 Value 列）。维度列用于分组和排序，指标列的值可以通过聚合函数 SUM、COUNT、MIN、MAX、REPLACE、HLL_UNION 和 BITMAP_UNION 等累加起来。因此，StarRocks 中的表也可以认为是多维的 Key 到多维指标的映射。

在 StarRocks 中，表的数据按列存储。物理上，一列数据会经过分块编码、压缩等操作，然后持久化存储到非易失设备上。但在逻辑上，一列数据可以看成是由相同类型的元素构成的一个数组。 一行数据的所有字段在各自的列数组中保持对齐，即拥有相同的数组下标，该下标称之为序号或者行号。该序号是隐式的，不需要存储。表中所有的行按照维度列，做多重排序，排序后的位置就是该行的行号。

查询时，如果指定了维度列的等值条件或者范围条件、并且这些条件中维度列可以构成表维度列的前缀，则可以利用数据的有序性，使用范围查找 (Range Scan) 快速锁定目标行。例如，表 `table1` 包含四列：`event_day`、`siteid`、`citycode` 和 `username`。假设要把这张表中的数据按照 `pv` 字段做聚合查询。如果查询条件为 `event_day > 2020-09-18 and siteid = 2`，则可以使用范围查找；如果查询条件为 `citycode = 4 and username in ["Andy", "Boby", "Christian", "StarRocks"]`，则无法使用范围查找。

## 稀疏索引

使用范围查找时，如何快速找到起始的目标行呢？答案是使用前缀索引 (Prefix Index)，如下图所示。

![img](https://starrocks.feishu.cn/space/api/box/stream/download/asynccode/?code=OGZlODJjODk3NjFkYjU0ODFmNDAyZjI0MzdhYjFlODBfb1lERGV5bjJyWHZKZzlhaFJ1N1pyV3dLc3JuSXF3Y25fVG9rZW46Ym94Y25tdWphcHZnV3djSWdSWDExYUdvTE5nXzE2NTY0NjkxNDc6MTY1NjQ3Mjc0N19WNA)

一张表中的数据组织主要由三部分构成：

- 前缀索引

表中每 1024 行数据构成一个逻辑数据块 (Data Block)。每个逻辑数据块在前缀索引表中存储一项索引，内容为表的维度列的前缀，并且长度不超过 36 字节。前缀索引是一种稀疏索引，使用某行数据的维度列的前缀查找索引表，可以确定该行数据所在逻辑数据块的起始行号。

- Per-column data block

表中每列数据都按 64 KB 分块存储。数据块作为一个单位单独编码、压缩，也作为 I/O 单位，整体写回设备或者读出。

- Per-column cardinal index

表中每列数据都有各自的行号索引表。 列的数据块和行号索引一一对应，索引由数据块的起始行号及数据块的位置和长度信息构成。用数据行的行号查找行号索引表，可以获取包含该行号的数据块所在的位置，读取目标数据块后，可以进一步查找数据。

由此可见，查找维度列的前缀的过程包含以下五个步骤：

1. 先查找前缀索引，获得逻辑数据块的起始行号。
2. 查找维度列的行号索引，定位到维度列的数据块。
3. 读取数据块。
4. 解压、解码数据块。
5. 从数据块中找到维度列前缀对应的数据项。

## 加速处理

StarRocks 通过如下机制实现数据的加速处理：

### 预先聚合

StarRocks 支持聚合模型，维度列取值相同的数据行可合并一行。合并后，数据行的维度列取值不变，指标列的取值为这些数据行的聚合结果。您需要给指标列指定聚合函数。通过预先聚合，可以加速聚合操作。

### 分区分桶

StarRocks 中，表被划分成多个 Tablet，每个 Tablet 多副本冗余存储在 BE 上。BE 和 Tablet 的数量可以根据计算资源和数据规模的变化而弹性伸缩。查询时，多台 BE 可以并行地查找 Tablet，从而快速获取数据。此外，Tablet 的副本可以复制和迁移，从而增强数据可靠性，并避免数据倾斜。总之，分区分桶有效保证了数据访问的高效性和稳定性。

### 物化视图

前缀索引可以加速数据查找，但是前缀索引依赖维度列的排列次序。如果使用非前缀的维度列构造查找谓词，则无法使用前缀索引。您可以为数据表创建物化视图。物化视图的数据组织和存储与数据表相同，但物化视图拥有自己的前缀索引。在为物化视图创建索引时，可指定择聚合的粒度、列的数量和维度列的次序，使频繁使用的查询条件能够命中相应的物化视图索引。

### 列级索引

StarRocks 支持布隆过滤器 (Bloom Filter)、ZoneMap 索引和 位图 (Bitmap) 索引等列级别的索引技术：

- 布隆过滤器有助于快速判断数据块中不含所查找的值。

- ZoneMap 索引有助于通过数据范围快速过滤出待查找的值。

- 位图索引有助于快速计算出枚举类型的列满足一定条件的行。
