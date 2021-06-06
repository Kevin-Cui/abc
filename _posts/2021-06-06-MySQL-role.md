---
layout: post
title: MySQL8.0 ROLE管理
categories: MySQL
description: MySQL8.0 ROLE管理
keywords: MySQL ROLE
---


数据库里对应的权限都可以指定赋予，那么角色的作用是什么？
数据库里角色是一个命名的权限集合，为了对许多拥有相似权限的用户进行分类管理，定义了角色的概念。与用户帐户一样，角色可以具有授予和撤消它们的特权。
比如：当多个用户分配复杂又细致的权限时，角色的作用就体现出来了。就是把一堆权限给一个角色，新用户只要使用这个角色，就能有对应的权限了。MySQL8.0里角色是怎样实现。

角色相关命令和配置方式：
1.命令接口：

命令	说明
CREATE ROLE and DROP ROLE	创建和删除角色
GRANT and REVOKE	是否激活角色
SHOW GRANTS	显示 账户/角色 所拥有的 权限或者角色
SET DEFAULT ROLE	设置账户默认使用什么角色
SET ROLE	改变当前会话的角色
CURRENT_ROLE()	显示当前会话的角色
WITH ADMIN OPTION	授予和撤销其他用户或角色
2.my.cnf配置参数：

参数	说明
mandatory_roles	允许定义用户登陆时强制权的角色
activate_all_roles_on_login	是否激活角色
角色和用户区别
1.不管创建用户和角色都是在mysql.user表里：
image.png
备注：区别在于account_locked，password_expired

2.查了对应的mysql库发现没有特别的role相关的表，那是否可以理解 role其实也是用户，只是没有密码和锁住无法登陆
```
ALTER USER 'role_developer'@'%' IDENTIFIED BY '123456';
ALTER USER 'role_developer'@'%' ACCOUNT UNLOCK;
```
备注：role账号，发现可以正常登陆。到这里可以大致理解，实际上角色和用户都是相等的。只是通过关系绑在一起。算是打破了常理，方式值得借鉴和学习。

看到这里，还是比较简单易懂的，大致都有了解。下面是练习SQL语句。

例子
#1.创建角色
```
mysql>
DROP ROLE IF EXISTS  'role_developer'@'%' ,'role_read'@'%' ,'role_write'@'%' ;
CREATE ROLE 'role_developer'@'%', 'role_read'@'%', 'role_write'@'%'          ;
```
#2.赋予权限:
```
mysql>
GRANT ALL ON world.* TO 'role_developer';
GRANT SELECT ON world.* TO 'role_read';
GRANT INSERT, UPDATE, DELETE ON world.* TO 'role_write';
```
3.创建用户;
```
mysql>
DROP USER IF EXISTS  'user_dev'@'%' , 'user_read'@'%' ,'user_write'@'%'    ;
CREATE USER 'user_dev'@'%'   IDENTIFIED BY '123456' ; 
CREATE USER 'user_read'@'%'  IDENTIFIED BY '123456' ; 
CREATE USER 'user_write'@'%' IDENTIFIED BY '123456' ; 
```
4.查询用户表状态
```
mysql>
SELECT  user,host,account_locked,password_expired 
FROM mysql.user WHERE user LIKE 'user_%' OR user LIKE 'role_%';
```
5.角色授予和撤销:
```
mysql>
GRANT 'role_developer'@'%' TO 'user_dev'@'%';
GRANT 'role_developer'@'%' TO 'user_read'@'%'  WITH ADMIN OPTION;
GRANT 'role_write'@'%'     TO 'user_write'@'%' WITH admin OPTION;
```
##回收角色
```
# REVOKE 'role_developer'@'%' FROM  'user_dev'@'%';
# REVOKE 'role_developer'@'%' FROM  'user_read'@'%';
# REVOKE 'role_write'@'%'     FROM  'user_write'@'%';
```
5.激活ROLE
```
mysql>
SET DEFAULT ROLE ALL TO  'user_dev'@'%';
SET DEFAULT ROLE ALL TO  'user_read'@'%';
SET DEFAULT ROLE ALL TO  'user_write'@'%';
可以用user账号登陆操作了。
```

6.ROLE操作
```
mysql>
SET ROLE NONE;                     #无角色 
SET ROLE ALL EXCEPT 'role_write';  #除已命名的角色外的所有角色            
SET ROLE ALL;                      #所有角色。
SELECT CURRENT_ROLE();             #当前角色  
```

7.强制给所有用户赋予角色，启动角色方式
```
mysql>
SET PERSIST mandatory_roles = 'role1,role2@%,r3@%.example.com';
SET PERSIST activate_all_roles_on_login = ON;
[msyqld]
mandatory_roles='role_developer'
activate_all_roles_on_login = ON
activate_all_roles_on_login：
```
服务器会在登录时激活每个帐户的所有角色。这优先于使用SET default ROLE指定的默认角色。对于在定义器上下文中执行的存储程序和视图，也只在开始执行时应用。

8.其他
ROLES_GRAPHML:返回utf8字符串xml(graphml)有用户信息，应该用户api接口扩展。

mysql>
SELECT ROLES_GRAPHML() 
总结：
便利用户分类管理，实际场景用的不多。
角色和用户是互可互通，有点意思。
参考：
https://dev.mysql.com/doc/refman/8.0/en/roles.html
