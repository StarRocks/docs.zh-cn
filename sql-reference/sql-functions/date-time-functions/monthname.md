# monthname

## 功能

返回日期对应的月份名字

### 语法

```Haskell
MONTHNAME(date)
```

## 参数说明

`date`: 支持的数据类型为 DATE 或 DATETIME 类型

## 返回值说明

返回值的数据类型为 VARCHAR

## 示例

```Plain Text
MySQL > select monthname('2008-02-03 00:00:00');
+----------------------------------+
| monthname('2008-02-03 00:00:00') |
+----------------------------------+
| February                         |
+----------------------------------+
```

## 关键词

MONTHNAME
