# minute

## description

### Syntax

```Haskell
INT MINUTE(DATETIME date)
```

获得日期中的分钟的信息，返回值范围从 0-59。

参数为 Date 或者 Datetime 类型

## example

```Plain Text
MySQL > select minute('2018-12-31 23:59:59');
+-----------------------------+
|minute('2018-12-31 23:59:59')|
+-----------------------------+
|                          59 |
+-----------------------------+
```

## keyword

MINUTE
