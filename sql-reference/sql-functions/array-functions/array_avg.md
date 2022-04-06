# array_avg

## 功能

求取一个 ARRAY 中的所有数据的平均数, 返回这个结果

## 语法

```Haskell
array_avg(array_type)
```

## 参数说明

`array_type`: 支持的数据类型有 BOOLEAN、TINYINT、SMALLINT、INT、BIGINT、LARGEINT、FLOAT、DOUBLE、DECIMALV2

## 返回值说明

当 `array_type` 类型为 BOOLEAN、TINYINT、SMALLINT、INT、BIGINT 、LARGEINT、FLOAT、DOUBLE 时返回值为 DOUBLE 类型

当 `array_type` 类型为 DECIMALV2 时返回值为 DECIMALV2 类型

## example

```plain text
mysql> select array_avg([11, 11, 12]);
+-----------------------+
| array_avg([11,11,12]) |
+-----------------------+
| 11.333333333333334    |
+-----------------------+

mysql> select array_avg([11.33, 11.11, 12.324]);
+---------------------------------+
| array_avg([11.33,11.11,12.324]) |
+---------------------------------+
| 11.588                          |
+---------------------------------+
```

## keyword

ARRAY_AVG, ARRAY
