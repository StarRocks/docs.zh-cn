# from_days

## 功能

通过距离 0000-01-01 日的天数计算出哪一天

## 语法

```Haskell
FROM_DAYS(n)
```

## 参数说明

`n`: 支持的数据类型为 INT 类型

## 返回值说明

返回值的数据类型为 DATE

## 示例

```Plain Text
MySQL > select from_days(730669);
+-------------------+
| from_days(730669) |
+-------------------+
| 2000-07-03        |
+-------------------+
```

## 关键词

FROM_DAYS, FROM, DAYS
