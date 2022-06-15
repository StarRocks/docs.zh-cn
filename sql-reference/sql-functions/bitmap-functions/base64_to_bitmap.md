# base64_to_bitmap

## 功能

导入数据到StarRocks过程中，会在StarRocks外部将bitmap算好，将bitmap作为一个变量导入进来。由于目前仅支持字符串导入，因此需要先对bitmap进行序列化与Base64编码，然后再导入。

## 语法

```sql
base64_to_bitmap(bitmap)
```

## 参数说明

`bitmap`：支持的数据类型为STRING，可由Java或者C++接口先创建BitmapValue对象，然后添加元素、序列化、Base64编码，得到的Base64字符串将作为函数入参。

## 示例

创建库表。

```sql
create database bitmapdb;
use bitmapdb;
CREATE TABLE `bitmap_table` (
  `tagname` varchar(65533) NOT NULL COMMENT "标签名称",
  `tagvalue` varchar(65533) NOT NULL COMMENT "标签值",
  `userid` bitmap NOT NULL COMMENT "访问用户ID"
) ENGINE=OLAP
PRIMARY KEY(`tagname`, `tagvalue`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`tagname`) BUCKETS 1
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT"
);
```

### Java demo

步骤1：将StarRocks压缩包中的FE模块**spark-dpp-1.0.0.jar**，存放到本地的Maven仓库对应目录，比如：**~/.m2/repository/com/starrocks/spark-dpp/1.0.0/**。

步骤2：创建Maven工程，将如下依赖添加到**pom.xml**文件中。

```java
<dependency>
    <groupId>com.starrocks</groupId>
    <artifactId>spark-dpp</artifactId>
    <version>1.0.0</version>
</dependency>
```

步骤3：创建BitmapValue对象，添加元素，调用序列化接口，进行base64编码。
返回的Base64字符串，就是`base64_to_bitmap`函数的入参。

```java
import com.starrocks.load.loadv2.dpp.BitmapValue;

import java.io.*;
import java.util.Base64;

public class BitmapBase64Test {
    public static String serialize() {
        String result = "";
        try {
            BitmapValue bitmapValues = new BitmapValue();
            bitmapValues.add(1);
            bitmapValues.add(2);
            bitmapValues.add(3);

            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            DataOutput output = new DataOutputStream(bos);
            bitmapValues.serialize(output);

            byte[] encode = Base64.getEncoder().encode(bos.toByteArray());
            result = new String(encode);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return result;
    }

    public static void main(String[] args) {
        String encode = serialize();

        // 通过使用 base64_to_bitmap 函数写入数据到bitmap_table表中。
        String isql = "insert into bitmapdb.bitmap_table values('持有产品','保险',base64_to_bitmap('" + encode + "'));";
        System.out.println(isql);

        // then insert into db use this sql
        // Example: insert into bitmapdb.bitmap_table values('持有产品','保险',base64_to_bitmap('AjowAAABAAAAAAACABAAAAABAAIAAwA='));
    }
}
```

### Stream Load demo

有JSON格式文件**simpledata**, 内容如下，其中编码后的`userid`来源于上面的Java示例。

- 文件内容

```JSON
{
    "tagname": "持有产品", "tagvalue": "保险", "userid":"AjowAAABAAAAAAACABAAAAABAAIAAwA="
}
```

- 导入JSON文件中的数据到`bitmap_table`。

```plain text
curl --location-trusted -u root: -H "columns: c1,c2,c3,tagname=c1,tagvalue=c2,userid=base64_to_bitmap(c3)" -H "label:bitmap123" -H "format: json" -H "jsonpaths: [\"$.tagname\",\"$.tagvalue\",\"$.userid\"]" -T simpleData http://0.0.0.0:8030/api/bitmapdb/bitmap_table/_stream_load
```

- 从`bitmap_table`表中查询数据。

```plain text
mysql> select tagname,tagvalue,bitmap_to_string(userid) from bitmap_table;
+--------------+----------+----------------------------+
| tagname      | tagvalue | bitmap_to_string(`userid`) |
+--------------+----------+----------------------------+、
| 持有产品      | 保险      | 1,2,3                      |
+--------------+----------+----------------------------+
1 rows in set (0.01 sec)
```
