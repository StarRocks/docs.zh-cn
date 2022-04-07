# to_days

## 功能

返回距离 0000-01-01 的天数

## 语法

```Haskell
TO_DAYS(date)
```

## 参数说明

`datetime`: 支持的数据类型 DATE 或 DATETIME 类型

## 返回值说明

返回值的数据类型为 INT

## 示例

```Plain Text
MySQL > select to_days('2007-10-07');
+-----------------------+
| to_days('2007-10-07') |
+-----------------------+
|                733321 |
+-----------------------+
```

## 关键词

TO_DAYS, TO, DAYS
