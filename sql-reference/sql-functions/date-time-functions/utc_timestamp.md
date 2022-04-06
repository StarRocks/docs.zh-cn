# utc_timestamp

## 功能

返回当前 UTC 日期和时间

## 语法

```Haskell
UTC_TIMESTAMP()
```

## 参数说明

无

## 返回值说明

返回值的数据类型为 DATETIME

## 示例

该函数用在字符串语境("YYYY-MM-DD HH: MM: SS")中和用在数字语境中("YYYYMMDDHHMMSS")返回的两种格式

```Plain Text
MySQL > select utc_timestamp(),utc_timestamp() + 1;
+---------------------+---------------------+
| utc_timestamp()     | utc_timestamp() + 1 |
+---------------------+---------------------+
| 2019-07-10 12:31:18 |      20190710123119 |
+---------------------+---------------------+
```

## 关键词

UTC_TIMESTAMP, UTC, TIMESTAMP
