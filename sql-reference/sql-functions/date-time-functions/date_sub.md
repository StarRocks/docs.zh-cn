# date_sub

## 功能

对给定日期减去指定的时间间隔

## 语法

```Haskell
DATE_SUB(date,INTERVAL expr type)
```

## 参数说明

`date`: 合法的日期表达式, 支持的数据类型为 DATETIME

`expr`: 期望减去的时间间隔

`type`: 可以是 YEAR, MONTH, DAY, HOUR, MINUTE, SECOND 中任一种值

## 返回值说明

返回值的数据类型为 DATETIME

## 示例

在 `2010-11-30 23:59:59` 日期基础上减两天

```Plain Text
MySQL > select date_sub('2010-11-30 23:59:59', INTERVAL 2 DAY);
+-------------------------------------------------+
| date_sub('2010-11-30 23:59:59', INTERVAL 2 DAY) |
+-------------------------------------------------+
| 2010-11-28 23:59:59                             |
+-------------------------------------------------+
```

## 关键词

DATE_SUB, DATE, SUB
