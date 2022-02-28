# array_concat

## 功能

拼接多个数组。

## 语法

```Haskell
output array_concat(input0, input1, ...)
```

## 参数说明

* input：输入为不限数量的相同类型的数组(input0, input1, ...)。

## 返回值说明

类型为Array(与输入input保持一致)，内容为将输入input(input0, input1, ...)，有序拼接后构成的数组。

## 示例

**示例一**:

```plain text
mysql> select array_concat([57.73,97.32,128.55,null,324.2], [3], [5]) as res;
+-------------------------------------+
| res                                 |
+-------------------------------------+
| [57.73,97.32,128.55,null,324.2,3,5] |
+-------------------------------------+
```

**示例二**:

```plain text
mysql> select array_concat(["sql","storage","execute"], ["Query"], ["Vectorized", "cbo"]);
+----------------------------------------------------------------------------+
| array_concat(['sql','storage','execute'], ['Query'], ['Vectorized','cbo']) |
+----------------------------------------------------------------------------+
| ["sql","storage","execute","Query","Vectorized","cbo"]                     |
+----------------------------------------------------------------------------+
```

**示例 三**:

```plain text
mysql> select array_concat(["sql",null], [null], ["Vectorized", null]);
+---------------------------------------------------------+
| array_concat(['sql',NULL], [NULL], ['Vectorized',NULL]) |
+---------------------------------------------------------+
| ["sql",null,null,"Vectorized",null]                     |
+---------------------------------------------------------+
```

## 关键字

ARRAY_CONCAT
