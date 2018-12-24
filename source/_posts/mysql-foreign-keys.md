---
title: mysql —— 外键
date: 2017-04-08 15:32:04
tags: mysql
---

<!-- more -->

> **作用：** 保持数据的完整性和一致性
> **环境:** mysql数据库目前只有InnoDB引擎支持外键
> **关键字：** `foreign_key`， `references`


## 语法


![](http://asset.eienao.com/17-3-18/93322072-file_1489851435340_66c1.png)

定义为 `foreign_key`的字段所在的表称为 **从表**，也叫做被约束的表，反之，`references` 参考的字段（必须为主键，或者唯一索引）所在的表称为**主表**，即约束表
        
**事件触发限制的取值**

当取值为`no action`或者`restrict`时，则当在主表（即外键的来源表）中删除对应记录时，首先检查该记录是否有对应外键，如果有则不允许删除。

当取值为`cascade`时，则当在主表（即外键的来源表）中删除对应记录时，首先检查该记录是否有对应外键，如果有则也删除外键在子表（即包含外键的表）中的记录。

当取值为Set Null时，则当在主表（即外键的来源表）中删除对应记录时，首先检查该记录是否有对应外键，如果有则设置子表中该外键值为null（不过这就要求该外键允许取null）。

---

## 例子
####mysql中的使用：
```sql
    //用户表 -> 档案表 ，一对一关系，添加外键
    //在profile表添加外键，则profile表为被约束表（从表）
    alter table `profile` add constraint `user_profile` foreign key (`user_id`) references `user`(`id`) on delete cascade on update cascade;
```
on delete cascade 当删除主表(user表)中的记录时,会对被删除的记录的id(references的字段)和从表(profile表)中的user_id(foreign key的字段)进行匹配,如果相同,则删除从表中对应的记录

on update在参考字段为主键id时并不常用,毕竟修改 主键id的需求并不是那么常见

####laravel中的使用：
```php
    //在迁移文件中创建外键索引
    $table->foreign('user_id')->references('id')->on('user') ->onDelete('cascade');
    
    //要在迁移文件中删除一个外键，可以使用 dropForeign 方法。外键约束和索引使用同样的命名规则——连接表名、外键名然后加上“_foreign”后缀：

    $table->dropForeign('profile_user_id_foreign');
```

## 补充

- 多对多表间关系中外键通常定义在中间表中
- 一对多关系中,外键通常定义在多的一方
- 一对一关系中,定义外键的一方称为从表,个人理解是,不需要像上述两种关系另起一个字段用来定义外键,从表的主键和主表的主键可以是同一个字段,并缔结外键索引关系, 例如上述的学生表和该学生的档案表的主键同时为 id (学生id),并使用外键约束

当然并不是存在关系的表就一定要使用外键索引，亦或是外键约束。通过php程序依旧可以保证数据的完整和一致。是否使用外键要从性能,灵活性等多方面进行考虑
