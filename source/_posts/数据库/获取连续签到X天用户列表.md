---
title: 获取连续签到X天用户列表
tags:
  - 数据库
originContent: >-
  近期需要做一个连续签到X天用户列表功能, 一开始没什么头绪，查了下发现有这个函数


  ### 数据库的 datediff 函数

  ```

  select datediff('2019-03-07','2019-02-27')

  ```

  运行结果是：

  ```

  datediff('2019-03-07','2019-02-27')

  8

  ```

  ### 数据表结构：

  ![image.png](/images/2019/03/07/14b4ca30-4098-11e9-9318-a5eab767ada4.png)


  ### 最终使用语句：

  ```sql

  select user_name,sign_date,IF(@pre=user_name and
  DATEDIFF(sign_date,@pre_date)=1,@rownum:=@rownum+1,@rownum:=1),
   
  @pre:=user_name,@pre_date:=sign_date
   from (
  select user_name,sign_date from user_sign

  GROUP BY user_name,sign_date ORDER BY user_name   ,sign_date  ) a ,(select
  @pre:='',@rownum:=0,@pre_date:='' ) b

  ```
categories:
  - 数据库
toc: false
date: 2019-03-07 17:17:11
---

近期需要做一个连续签到X天用户列表功能, 一开始没什么头绪，查了下发现有这个函数

### 数据库的 datediff 函数
```
select datediff('2019-03-07','2019-02-27')
```
运行结果是：
```
datediff('2019-03-07','2019-02-27')
8
```
### 数据表结构：
![image.png](/images/2019/03/07/14b4ca30-4098-11e9-9318-a5eab767ada4.png)

### 最终使用语句：
```sql
select user_name,sign_date,IF(@pre=user_name and DATEDIFF(sign_date,@pre_date)=1,@rownum:=@rownum+1,@rownum:=1),
 
@pre:=user_name,@pre_date:=sign_date
 from (
select user_name,sign_date from user_sign
GROUP BY user_name,sign_date ORDER BY user_name   ,sign_date  ) a ,(select @pre:='',@rownum:=0,@pre_date:='' ) b
```