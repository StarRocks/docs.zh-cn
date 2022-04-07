# array_sum

## 功能

对一个 ARRAY 中的所有数据做和, 返回这个结果

## 语法

```Haskell
array_sum(array(type))
```

## 参数说明

`type`: 支持的数据类型有 BOOLEAN、TINYINT、SMALLINT、INT、BIGINT、LARGEINT、FLOAT、DOUBLE、DECIMALV2

## 返回值说明

当 `type` 类型为 BOOLEAN、TINYINT、SMALLINT、INT、BIGINT 时返回值为 BIGINT 类型

当 `type` 类型为 LARGEINT 时返回值为 LARGEINT 类型

当 `type` 类型为 FLOAT、DOUBLE 时返回值为 DOUBLE 类型

当 `type` 类型为 DECIMALV2 时返回值为 DECIMALV2 类型

## 示例

```plain text
mysql> select array_sum([11, 11, 12]);
+-----------------------+
| array_sum([11,11,12]) |
+-----------------------+
| 34                    |
+-----------------------+

mysql> select array_sum([11.33, 11.11, 12.324]);
+---------------------------------+
| array_sum([11.33,11.11,12.324]) |
+---------------------------------+
| 34.764                          |
+---------------------------------+
```

## 关键词

ARRAY_SUM, ARRAY
