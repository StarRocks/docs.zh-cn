# timestampadd

## 功能

将整数表达式间隔添加到日期或日期时间表达式 datetime_expr 中

## 语法

```Haskell
TIMESTAMPADD(unit, interval, datetime_expr)
```

## 参数说明

`unit`: 可以是 YEAR, MONTH, DAY, HOUR, MINUTE, SECOND, WEEK 中任一种值

`interval`: 整数表达式, 支持 INT 类型

`datetime_expr`: 日期或日期时间表达式, 支持的数据类型为 DATE 或 DATETIME 类型

## 返回值说明

返回值的数据类型为 DATETIME

## 示例

```plain text

MySQL > SELECT TIMESTAMPADD(MINUTE,1,'2019-01-02');
+------------------------------------------------+
| timestampadd(MINUTE, 1, '2019-01-02 00:00:00') |
+------------------------------------------------+
| 2019-01-02 00:01:00                            |
+------------------------------------------------+

MySQL > SELECT TIMESTAMPADD(WEEK,1,'2019-01-02');
+----------------------------------------------+
| timestampadd(WEEK, 1, '2019-01-02 00:00:00') |
+----------------------------------------------+
| 2019-01-09 00:00:00                          |
+----------------------------------------------+
```

## 关键词

TIMESTAMPADD
