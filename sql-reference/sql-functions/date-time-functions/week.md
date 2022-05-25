# week

## description

### Syntax

```Haskell
INT WEEK(DATETIME/DATE date, INT mode)
```
与mysql中week函数的语义相同，根据mode返回date所在，是属于该年中的第几个周。
mode的取值范围是[0-7], 具体的语义由如下表格指定:

```Plain Text
Mode	First day of week	Range	Week 1 is the first week …
0	Sunday	                0-53	with a Sunday in this year
1	Monday	                0-53	with 4 or more days this year
2	Sunday	                1-53	with a Sunday in this year
3	Monday	                1-53	with 4 or more days this year
4	Sunday	                0-53	with 4 or more days this year
5	Monday	                0-53	with a Monday in this year
6	Sunday	                1-53	with 4 or more days this year
7	Monday	                1-53	with a Monday in this year
```

## example

对于‘2007-01-01’, 经过查询得知是周一。

那么mode=0, 可以得到结果为0，此时周日是作为一周的第一天，‘2007-01-01’不能作为第一周。
```Plain Text
mysql> SELECT WEEK('2007-01-01', 0);
+-----------------------+
| week('2008-02-20', 0) |
+-----------------------+
|                     0 |
+-----------------------+
1 row in set (0.02 sec)
```

mode=1，得到结果为1，此时周一作为了一周的第一天。
```Plain Text
mysql> SELECT WEEK('2007-01-01', 1);
+-----------------------+
| week('2007-01-01', 1) |
+-----------------------+
|                     1 |
+-----------------------+
1 row in set (0.02 sec)
```

mode=2, 得到结果53，此刻同样周日作为一周的第一天，但是取值范围是[1-53]，所以返回53，表示这是上一年的最后一周。
```Plain Text
mysql> SELECT WEEK('2007-01-01', 2);
+-----------------------+
| week('2007-01-01', 2) |
+-----------------------+
|                    53 |
+-----------------------+
1 row in set (0.01 sec)
```
## keyword

WEEK
