# bitmap_union_count

## 功能

计算输入的一组 bitmap 值的并集，并返回并集的基数。

## 语法

```sql
BITMAP_UNION_COUNT(value)
```

### 参数说明

- `value` ：输入的一组 bitmap 值，支持的数据类型为 BITMAP。

## 返回值说明

返回值的数据类型为 BIGINT。

## 示例

使用该函数计算网页的 PV 数据。假设 `user_id` 字段类型为 INT，下面两个查询是等价的。

```sql
select page_id, bitmap_union_count(to_bitmap(user_id))
from table
group by page_id;
```

```sql
select page_id, count(distinct user_id)
from table
group by page_id;
```
