# EXECUTE AS

## 功能

在使用 [GRANT](../account-management/GRANT.md) 语句得到以指定用户身份执行操作的权限后，您可以使用 EXECUTE AS 语句将当前会话的执行上下文切换到该指定用户。

## 语法

```SQL
EXECUTE AS user WITH NO REVERT;
```

## 参数说明

`user`：该指定用户必须为已存在的用户。

## 注意事项

- 当前登录用户（即调用 EXECUTE AS 语句的用户）必须有 IMPERSONATE 到指定用户的权限。

- EXECUTE AS 语句必须包括 WITH NO REVERT 子句，表示在当前会话结束前，当前会话的执行上下文不能切换为原登录用户。

## 示例

将会话的执行上下文切换到用户 `test2`。

```SQL
execute as test2 with no revert;
```

语句执行成功后，可以通过 `select current_user()` 命令获取当前用户。

```undefined
select current_user();
+-----------------------------+
| CURRENT_USER()              |
+-----------------------------+
| 'default_cluster:test2'@'%' |
+-----------------------------+
1 row in set (0.03 sec)
```

## Description

After you use the GRANT statement to impersonate a specific user identity to perform operations, you can use the EXECUTE AS statement to switch the execution context of the current session to this user.

## Syntax

```SQL
EXECUTE AS user WITH NO REVERT;
```

## Parameters

`user`: The user must already exist.

## Usage notes

- The current login user (who calls the EXECUTE AS statement) must be granted the IMPERSONATE permission.

- The EXECUTE AS statement must contain the WITH NO REVERT clause, which means the execution context of the current session cannot be switched back to the original login user before the current session ends.

## Examples

Switch the execution context of the current session to the user `test2`.

```SQL
MySQL [(none)]> execute as test2 with no revert;
Query OK, 0 rows affected (0.01 sec)
```

After the switch succeeds, you can run the `select current_user()` command to obtain the current user.

```SQL
MySQL [(none)]> select current_user();
+-----------------------------+
| CURRENT_USER()              |
+-----------------------------+
| 'default_cluster:test2'@'%' |
+-----------------------------+
1 row in set (0.03 sec)
```
