# perecentile_approx_raw

## 功能

限制 `percentile` 的分位值

## 语法

```Haskell
PERCENTILE_APPROX_RAW(x,y);
```

## 参数说明

`x`: 支持的数据类型为 PERCENTILE

`y`: 支持的数据类型为 DOUBLE

## 返回值说明

返回值的数据类型为 PERCENTILE

## 示例

```Plain Text
mysql> select percentile_approx_raw(percentile_hash(234.234), 1);
+----------------------------------------------------+
| percentile_approx_raw(percentile_hash(234.234), 1) |
+----------------------------------------------------+
|                                 234.23399353027344 |
+----------------------------------------------------+
1 row in set (0.01 sec)
```

## 关键词

PERCENTILE_APPROX_RAW
