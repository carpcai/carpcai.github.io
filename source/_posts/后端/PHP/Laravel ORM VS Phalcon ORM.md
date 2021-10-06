---
title: Laravel ORM VS Phalcon ORM
tags: []
categories:
  - 后端
toc: false
date: 2020-06-18 07:00:17
---

## 前言

由于php5逐步停止维护，公司由于技术发展得需要，需要从只支持php5的CI2框架，迁移到支持php7以上的高性能phalcon框架，本来是一件非常美好的事情，但迁移后无奈phalcon的原生ORM实在太难用了，希望引入更加易用的第三方ORM工具，于是开始了以下调研之旅。

## 调研结果

Laravel ORM 开发效率高(原本同事写了2天的代码2小时重构完成)
Phalcon 原生ORM性能强， 做压力测试50次大数据量传输请求，laravel ORM耗时50秒，phalcon 17秒。
性能差距甚大，不由感叹phalcon框架性能强大。

## 以下面这段代码为例测试性能：

### 代码：

#### 原生phalcon 操作

```php
<?php
echo "内存初始状态：".$this->format_bytes(memory_get_usage()) . '
';
        
//Phalcon 查询10条数据
$res = Boxes::query()->createBuilder()
    ->limit(10)
    ->getQuery()
    ->execute()
    ->toArray();

echo "查询结束内存状态：".$this->format_bytes(memory_get_usage());
```

#### Laravel ORM 操作

```php
<?php
echo "内存初始状态：".$this->format_bytes(memory_get_usage()) . '
';

//Laravel ORM 查询10条数据
$res = Boxes::query()->limit(10)->get()->toArray();

echo "查询结束内存状态：".$this->format_bytes(memory_get_usage());
```

```php
<?php
echo "内存初始状态：".$this->format_bytes(memory_get_usage()) . '
';

//Laravel ORM 查询10条数据
$res = $this->mysql::select('select * from boxes limit 100');

echo "查询结束内存状态：".$this->format_bytes(memory_get_usage());
```

### 耗时情况比较

在Phalcon框架原生query builder:
![](https://cdn.carpcai.cn/localfile/c99ae21e9613b42aabef97bb48f98f1f.jpg)

在Phalcon框架中引入  Laravel Eloquent ORM:
![](https://cdn.carpcai.cn/localfile/c8d2c8d201b96ae5bedea91cf8ad35a9.jpg)

在Phalcon框架中使用 Laravel Eloquent ORM 的原生sql操作:
![](https://cdn.carpcai.cn/localfile/7e4cbc9031c9cf2dba09f93429767e2e.jpg)

由 上图可见，引入ORM之后，查询性能明显比原生ORM延迟近3倍(原生ORM14秒，Laravel ORM 36秒)

### 内存使用情况

#### 原生phalcon 查询10条数据操作

```
内存初始状态：1.21 MB
查询结束内存状态：1.3 MB
```

#### Laravel ORM 查询10条数据操作

```
内存初始状态：1.2 MB
查询结束内存状态：5.54 MB
```

#### 原生phalcon 查询100条数据查询操作

```
内存初始状态：1.21 MB
查询结束内存状态：1.47 MB
```

#### Laravel ORM 查询100条数据操作

```
内存初始状态：1.2 MB
查询结束内存状态：5.72 MB
```

#### Laravel ORM 使用原生sql语句 查询100条数据操作

```
内存初始状态：1.2 MB
查询结束内存状态：2.83 MB
```

## 观察一下两个ORM请求到sql的语句分别有什么

### 接下来我们测试一下100次请求中，向mysql请求的次数（包括数据库连接，选择数据库，查询，退出数据库）：

1. Laravel ORM ：1351
2. Phalcon 原生: 1351，
   也就是不是数据库查询次数的原因。

```
set names 'utf8' collate 'utf8_unicode_ci'

select * from `tickets` limit 10

use `owdile001`

root@120.111.11.111 on owdile001 using TCP/IP
```

```
SELECT `tickets`.`id`, `tickets`.`xxx_id`, `tickets`.`type` FROM `tickets` LIMIT 10

DESCRIBE `tickets`

SET NAMES utf8

root@120.111.11.111 on owdile001 using TCP/IP
```

正常用户体验：

> 模拟两个用户同时使用,一共请求10次，每次请求存在两次sql查询

原生

```
Total time passed:		    2.17s
Avg time per request:		373.80ms
Requests per second:		4.62
Median time per request:	344.23ms
99th percentile time:		315.96ms
Slowest time for request:	480.00ms
```

Eloquent

```
Total time passed:		4.11s
Avg time per request:		706.96ms
Requests per second:		2.43
Median time per request:	666.99ms
99th percentile time:		574.17ms
Slowest time for request:	871.00ms
```

