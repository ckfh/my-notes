# 读书笔记

## 第3章：使用MySQL

```SQL
# 显示字段信息
SHOW COLUMNS FROM customers;
# MySQL支持用DESCRIBE作为上述的一种快捷方式
DESCRIBE customers;
# 显示广泛的服务器状态信息
SHOW STATUS;
# 显示创建数据库的MySQL语句
SHOW CREATE DATABASE learn_mysql;
# 显示创建表的MySQL语句
SHOW CREATE TABLE customers;
# 显示授予用户的安全权限
SHOW GRANTS;
# 显示服务器错误消息
SHOW ERRORS;
# 显示服务器警告消息
SHOW WARNINGS;
# 在命令行中执行可以显示允许的SHOW语句
HELP SHOW;
```

## 第4章：检索数据

```SQL
# 返回不同（唯一）的行
# 无法部分使用DISTINCT关键字，它将应用于所有列而不仅是前置它的列
SELECT DISTINCT vend_id
FROM products;
# 从第1行开始返回5行数据
# 注意数据库行索引从0开始
SELECT prod_name
FROM products
LIMIT 5;
# 从第5行开始返回5行数据
SELECT prod_name
FROM products
LIMIT 5, 5;
# MySQL5支持LIMIT的另一种替代语法
SELECT prod_name
FROM products
LIMIT 5 OFFSET 5;
# 使用完全限定列名以及完全限定表名，有一些情形需要完全限定名
SELECT products.prod_name
FROM learn_mysql.products;
```

## 第5章：排序检索数据

关系数据库设计理论认为，如果不明确规定排序顺序，则不应该假定检索出的数据的顺序有意义。

**MySQL和大多数数据库管理系统默认A被视为与a相同。**数据库管理员能够在需要时改变这种行为，但是你无法依赖ORDER BY子句来改变这种排序顺序。

```SQL
# 通常，ORDER BY子句中使用的列将是为显示所选择的列
# 但是，实际上用非检索的列排序数据是完全合法的
SELECT prod_name
FROM products
ORDER BY prod_name;
# 先按价格再按名称进行排序
SELECT prod_id, prod_price, prod_name
FROM products
ORDER BY prod_price, prod_name;
# 数据排序默认为升序，先按价格进行降序再按名称进行升序
# 如果都要降序，则每个列都需要指定DESC关键字
SELECT prod_id, prod_price, prod_name
FROM products
ORDER BY prod_price DESC, prod_name;
# 使用ORDER BY和LIMIT的组合找出最昂贵物品信息
# ORDER BY必须在FROM之后，LIMIT必须位于ORDER BY之后
SELECT prod_id, prod_price, prod_name
FROM products
ORDER BY prod_price DESC
LIMIT 1;
```

## 第6章：过滤数据
