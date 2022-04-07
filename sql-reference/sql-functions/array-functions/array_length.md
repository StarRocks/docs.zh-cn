# array_length

## 功能

返回数组中元素个数, 结果类型是 INT, 如果参数是 NULL, 结果也是 NULL

## 语法

```Haskell
array_length(any_array)
```

## 参数说明

`any_array`: 原数组, 支持的数据类型为 ARRAY 类型

## 返回值说明

返回值的数据类型为 INT

## 示例

```plain text
mysql> select array_length([1,2,3]);
+-----------------------+
| array_length([1,2,3]) |
+-----------------------+
|                     3 |
+-----------------------+
1 row in set (0.00 sec)

mysql> select array_length([[1,2], [3,4]]);
+-----------------------------+
| array_length([[1,2],[3,4]]) |
+-----------------------------+
|                           2 |
+-----------------------------+
1 row in set (0.01 sec)
```

## 关键词

ARRAY_LENGTH, ARRAY
