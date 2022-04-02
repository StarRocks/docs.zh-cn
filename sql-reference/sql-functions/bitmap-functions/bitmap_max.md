# BITMAP_MAX

## 功能

获取Bitmap中的最大值。如果Bitmap为NULL，则返回NULL。如果Bitmap为空，默认返回0。

## 语法

`BITMAP_MAX(bitmap_expr)`

## 参数说明

`bitmap_expr`：Bitmap的表达式，可以由 BITMAP_FROM_STRING 等函数构造。

## 返回值说明

返回 BIGINT 类型的值。

## 示例

```Plain%20Text
MySQL > select bitmap_max(bitmap_from_string("0, 1, 2, 3"));
+-------------------------------------------------+
|    bitmap_max(bitmap_from_string("0, 1, 2, 3")) |
+-------------------------------------------------+
|                         3                       |
+-------------------------------------------------+

MySQL > select bitmap_max(bitmap_from_string("-1, 0, 1, 2"));
+-------------------------------------------------+
|   bitmap_max(bitmap_from_string("-1, 0, 1, 2")) |
+-------------------------------------------------+
|                      NULL                       |
+-------------------------------------------------+

MySQL > select bitmap_max(bitmap_empty());
+----------------------------------+
|       bitmap_max(bitmap_empty()) |
+----------------------------------+
|                  0               |
+----------------------------------+
```

## 关键词

BITMAP_MAX, BITMAP
