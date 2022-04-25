# base64_to_bitmap

## 功能

导入数据到StarRocks过程中，在StarRocks外部将bitmap算好，将bitmap作为一个变量导入进来；由于目前只支持字符串导入，因此需要先对bitmap进行序列化与base64编码。
然后再导入。

## 语法

`base64_to_bitmap(bitmap)`

## 参数说明

`bitmap`：支持的数据类型为string，可由java或者c++接口先创建BitmapValue对象，然后添加元素、序列化、base64编码，得到的base64字符串将作为函数参数。

## 示例

```创建库、表命令：
create database bitmapdb;
use bitmapdb;
CREATE TABLE `bitmap_table` (
  `tagname` varchar(65533) NOT NULL COMMENT "标签名称",
  `tagvalue` varchar(65533) NOT NULL COMMENT "标签值",
  `userid` bitmap NOT NULL COMMENT "访问用户id"
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

### java demo

#### 步骤1

将StarRocks压缩包fe模块的spark-dpp-1.0.0.jar，放到本地的maven仓库对应目录，比如：~/.m2/repository/com/starrocks/spark-dpp/1.0.0/

#### 步骤2

```创建maven工程，将依赖添加到pom.xml中
<dependency>
    <groupId>com.starrocks</groupId>
    <artifactId>spark-dpp</artifactId>
    <version>1.0.0</version>
</dependency>
```

#### 步骤3

创建BitmapValue对象，添加元素，调用序列化接口，base64编码。返回的base64字符串，就是base64_to_bitmap函数的入参。

```如下java demo
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

        // insert or stream load, need use this function: base64_to_bitmap
        String isql = "insert into bitmapdb.bitmap_table values('持有产品','保险',base64_to_bitmap('" + encode + "'));";
        System.out.println(isql);

        // then insert into db use this sql
        // .... .......
    }
}
```

### stream load

json格式文件simpledata, 内容如下，其中编码后的userid来源于上面的java示例:

```simpleData内容
{
    "tagname": "持有产品", "tagvalue": "保险", "userid":"AjowAAABAAAAAAACABAAAAABAAIAAwA="
}
```

```导入命令
curl --location-trusted -u root: -H "columns: c1,c2,c3,tagname=c1,tagvalue=c2,userid=base64_to_bitmap(c3)" -H "label:bitmap123" -H "format: json" -H "jsonpaths: [\"$.tagname\",\"$.tagvalue\",\"$.userid\"]" -T simpleData http://0.0.0.0:8030/api/bitmapdb/bitmap_table/_stream_load
```

```查询命令
mysql> select tagname,tagvalue,bitmap_to_string(userid) from bitmap_table;
+--------------+----------+----------------------------+
| tagname      | tagvalue | bitmap_to_string(`userid`) |
+--------------+----------+----------------------------+、
| 持有产品      | 保险      | 1,2,3                      |
+--------------+----------+----------------------------+
1 rows in set (0.01 sec)
```

## 关键词

BASE64_TO_BITMAP, BITMAP, BASE64
