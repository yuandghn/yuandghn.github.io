---
layout:     post
title:      "Cannot create a JSON value from a string with CHARACTER SET 'binary'"
subtitle:   ""
date:       2019-08-01 13:50:00
author:     "Echo Yuan"
tags:
    - MySQL
    - JSON
---
项目里有使用到MySQL的JSON Data Type，偶尔会有从这个库导出Insert语句然后再导入到另一个库的需求，不过这个JSON类型的字段导出后是有问题的
```
Cannot create a JSON value from a string with CHARACTER SET 'binary'.
```
解决方法是先在文本编辑器里做个正则替换，把`(X'[^,\)]*')`替换成`CONVERT($1 using utf8mb4)`。举例如下：
```sql
INSERT INTO json_table (json_column) VALUES (X'7B22666F6F223A2022626172227D');
```
                                     ↓
```
INSERT INTO json_table (json_column) VALUES (CONVERT(X'7B22666F6F223A2022626172227D' using utf8mb4));
```
这样再导入就不会有问题了。

参考自：[https://stackoverflow.com/questions/38078119/mysql-5-7-12-import-cannot-create-a-json-value-from-a-string-with-character-set](https://stackoverflow.com/questions/38078119/mysql-5-7-12-import-cannot-create-a-json-value-from-a-string-with-character-set)