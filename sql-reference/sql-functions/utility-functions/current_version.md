# current_version

## 功能

获取当前 StarRocks 的版本

## 语法

```Haskell
current_version();
```

## 参数说明

无

## 返回值说明

返回值的数据类型为 VARCHAR

## 示例

```Plain Text
mysql> select current_version();
+-------------------+
| current_version() |
+-------------------+
| 2.1.2 0782ad7     |
+-------------------+
1 row in set (0.00 sec)
```

## 关键词

CURRENT_VERSION
