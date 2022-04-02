# array_append

## 功能

在数组末尾添加一个新的元素

## 语法

```Haskell
array_append(any_array, any_element)
```

## 参数说明

`any_array`: 原数组, 支持的数据类型为 ARRAY 类型

`any_element`: 要添加的目标元素, 可为 NULL，支持数据类型与 `any_array` 内元素类型一致

## 返回值说明

返回值的数据类型为 ARRAY

## example

```plain text
mysql> select array_append([1, 2], 3);
+------------------------+
| array_append([1,2], 3) |
+------------------------+
| [1,2,3]                |
+------------------------+
1 row in set (0.00 sec)

```


```plain text
mysql> select array_append([1, 2], NULL);
+---------------------------+
| array_append([1,2], NULL) |
+---------------------------+
| [1,2,NULL]                |
+---------------------------+
1 row in set (0.01 sec)

```

## keyword

ARRAY_APPEND, ARRAY
