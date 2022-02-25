# array_intersect 

## 功能

对于多个同类型数组，返回交集。

## 语法

```Haskell
output array_overlap(input)
```

## 参数说明

* input：不限数量的相同类型数组(input0, input1, ...)。

## 返回值说明

与input保持一致的相同类型数组，内容为所有输入数组(input0, input1, ...)的交集。

## 示例

**示例一**:

```plain text
mysql> SELECT array_intersect(["SQL", "storage"], ["mysql", "query", "SQL"], ["SQL"])
AS no_intersect ;
+--------------+
| no_intersect |
+--------------+
| ["SQL"]      |
+--------------+
```

**示例二**:

```plain text
mysql> SELECT array_intersect(["SQL", "storage"], ["mysql", null], [null]) AS no_intersect ;
+--------------+
| no_intersect |
+--------------+
| []           |
+--------------+
```

**示例 三**:

```plain text
mysql> SELECT array_intersect(["SQL", null, "storage"], ["mysql", null], [null]) AS no_intersect ;
+--------------+
| no_intersect |
+--------------+
| [null]       |
+--------------+
```

## 关键字

ARRAY_INTERSECT
