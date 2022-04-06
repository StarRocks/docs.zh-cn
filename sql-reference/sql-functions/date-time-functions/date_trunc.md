# date_trunc

## 背景

客户在使用时间函数时经常有关于 group by hour / week 这样的需求, 而且希望能够第一列直接展示成时间的格式, 希望有类似 oracle 的 trunc 函数这样的方式来对 datetime 进行高效的截断, 从而避免写出类似如下的低效 sql:

```sql
select ADDDATE(DATE_FORMAT(DATE_ADD(from_unixtime(`timestamp`), INTERVAL 8 HOUR),
                           '%Y-%m-%d %H:%i:00'),
               INTERVAL 0 SECOND),
    count()
from xxx group 1 ;
```

## 动机

为了更快实现对时间数据的截断, 提供了 date_trunc 系列函数

原本我们已经有了 year/month/day 等直接截取部分的时间函数

date_trunc 与此类似, 对 datetime 进行高位截断:

```Plain Text
date_trunc("minute", datetime):
2020-11-04 11:12:19 => 2020-11-04 11:12:00
```

## 函数签名

1. Oracle 使用的是 `TRUNC(date,[fmt])` 的函数格式

2. PostgreSQL/redshift 等使用的是 `date_trunc(text,time)` 的函数格式

3. StarRocks 采用 `date_trunc([fmt], datetime)` 的函数格式

## 函数实现

对于 date_trunc 的实现, 可以简单地分两步做:

1. 把 datetime 拆解成各个部分(年月日/是分秒), 然后提取所需要的部分

2. 根据所需要的部分组合为一个新的 datetime

对于 week/quarter 的情况需要做特殊计算

需要研究下 snowflake 的 week

```SQL
-- snowflake 中可以设置一周的第一天

-- 下面两个和默认一样把周一作为一周第一天
alter session set week_start = 0
alter session set week_start = 1

-- 把周三作为一周的第一天
alter session set week_start = 3
```

## 功能

将 `datetime` 按照 `fmt` 格式进行截断

## 语法

```Haskell
date_trunc(fmt, datetime)
```

## 参数说明

`fmt`: 支持字符串常量, 但是必须为固定的几个值(second, minute, hour, day, month, year, week, quarter), 输入不对的值在 FE 解析时直接返回错误信息, 支持的数据类型为 VARCHAR

`datetime`: 输入常量的情况会被正确识别到, 只处理一次, 支持的数据类型为 DATETIME

|  fmt 格式字符串   |  格式字符串语义   |  对应的例子  |
| --- | --- | --- |
| second |  截断到秒作为有效时间   |  2020-10-25 11:15:32 => 2020-10-25 11:15:32  |
| minute | 截断到分钟作为有效时间 | 2020-11-04 11:12:19 => 2020-11-04 11:12:00 |
| hour | 截断到小时作为有效时间 | 2020-11-04 11:12:13 => 2020-11-04 11:00:00 |
| day | 截断到天作为有效时间 | 2020-11-04 11:12:05 => 2020-11-04 00:00:00 |
| month | 截断到当月第一天作为有效时间 | 2020-11-04 11:12:51 => 2020-11-01 00:00:00 |
| year | 截断到当年第一天作为有效时间 | 2020-11-04 11:12:00 => 2020-01-01 00:00:00 |
| week | 截断到这个星期第一天作为有效时间 | 2020-11-04 11:12:00 => 2020-11-02 00:00:00 |
| quarter | 截断到这个季度第一天作为有效时间 | 2020-06-23 11:12:00 => 2020-04-01 00:00:00 |

## 返回值说明

返回值的数据类型为 DATETIME

## 示例

```Plain Text
MySQL > select date_trunc("hour", "2020-11-04 11:12:13");
+-------------------------------------------+
| date_trunc('hour', '2020-11-04 11:12:13') |
+-------------------------------------------+
| 2020-11-04 11:00:00                       |
+-------------------------------------------+
```

## 关键词

DATE_TRUNC, DATE
