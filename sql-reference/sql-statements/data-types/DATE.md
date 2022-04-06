# DATE

## 描述

### DATE 类型

日期类型，目前的取值范围是 ['0000-01-01', '9999-12-31'], 默认的打印形式是'YYYY-MM-DD'

### 示例

创建表时指定字段类型为 DATE

```sql
CREATE TABLE dateDemo (
    pk         INT     COMMENT "range [-2147483648, 2147483647]",
    make_time  DATE    NOT NULL COMMENT "YYYY-MM-DD"
) ENGINE=OLAP 
DUPLICATE KEY(pk)
COMMENT "OLAP"
DISTRIBUTED BY HASH(pk) BUCKETS 4;
```

### DATE 函数

语义：DATE(expr)

作用：将输入的类型转化为 DATE 类型

```sql
mysql> SELECT DATE('2003-12-31 01:02:03');
-> '2003-12-31'
```

除 DATE 函数外，StarRocks 还支持 YEAR()，MONTH()，DATE_FORMAT(expr,'%Y-%m-%d %T') 等日期函数。

## 关键字 (keywords)

DATE
