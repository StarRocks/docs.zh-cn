# unhex

## 功能

将输入的参数 `x`(16 进制字符)转化成原来的字符

## 语法

```Haskell
UNHEX(x);
```

## 参数说明

`x`: 支持的数据类型为 VARCHAR

## 返回值说明

返回值的数据类型为 VARCHAR

## 示例

```Plain Text
mysql> select unhex('3132');
+---------------+
| unhex('3132') |
+---------------+
| 12            |
+---------------+
1 row in set (0.00 sec)
```

## 关键词

UNHEX
