# aes_encrypt

## 功能

将参数加密并返回一个二进制字符串

## 语法

```Haskell
aes_encrypt(x,y);
```

## 参数说明

`x`: 支持的数据类型为 VARCHAR，它是加密的字符串

`y`: 支持的数据类型为 VARCHAR，它是用于加密参数 `x` 的 key 字符串

## 返回值说明

返回值的数据类型为 VARCHAR

## 示例

```Plain Text
mysql> select AES_ENCRYPT("star","rocks");
+------------------------------+
| aes_encrypt('star', 'rocks') |
+------------------------------+
| (𠱃0/񅬏                    |
+------------------------------+
1 row in set (0.02 sec)
```

## 关键词

AES_ENCRYPT
