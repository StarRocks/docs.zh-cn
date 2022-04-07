# curdate

## 功能

获取当前的日期

## 语法

```Haskell
CURDATE()
```

## 参数说明

无

## 返回值说明

返回值的数据类型为 DATE

## 示例

在字符串上下文中使用 CURDATE()函数

```Plain Text
MySQL > SELECT CURDATE();
+------------+
| CURDATE()  |
+------------+
| 2022-03-21 |
+------------+
```

在数字上下文中使用 CURDATE()函数

```Plain Text
MySQL > SELECT CURDATE() + 0;
+---------------+
| CURDATE() + 0 |
+---------------+
|      20220321 |
+---------------+
```

## 关键词

CURDATE
