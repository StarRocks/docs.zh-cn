# array_remove

## 功能

从数组中移除指定元素

## 语法

```Haskell
array_remove(any_array, any_element)
```

## 参数说明

`any_array`: 原数组, 支持的数据类型为 ARRAY 类型

`any_element`: 目标元素, 支持数据类型与 `any_array` 包含元素类型一致

## 返回值说明

返回值的数据类型为 ARRAY

## 示例

```plain text
mysql> select array_remove([1,2,3,null,3], 3);
+---------------------------------+
| array_remove([1,2,3,NULL,3], 3) |
+---------------------------------+
| [1,2,null]                      |
+---------------------------------+
1 row in set (0.01 sec)
```

## 关键词

ARRAY_REMOVE, ARRAY
