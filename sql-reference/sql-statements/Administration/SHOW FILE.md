# SHOW FILE

## 功能

该语句用于展示一个 database 内创建的文件。

## 语法

```sql
SHOW FILE [FROM database];
```

命令返回结果说明：

``` plain text
FileId:     文件ID，全局唯一
DbName:     所属数据库名称
Catalog:    自定义分类
FileName:   文件名
FileSize:   文件大小，单位字节
MD5:        文件的 MD5
```

## 示例

1. 查看数据库 my_database 中已上传的文件

    ```sql
    SHOW FILE FROM my_database;
    ```

## 关键字(keywords)

SHOW，FILE
