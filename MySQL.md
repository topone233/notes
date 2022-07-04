## 索引

### 索引类型

- index 普通索引

- unique 唯一索引

- primary key 主键索引

  事实上，PRIMARY KEY索引就是一个具有名称为PRIMARY的UNIQUE索引。这表示一个表只能包含一个PRIMARY KEY，因为一个表中不可能具有两个同名的索引。

### 创建索引

#### 1. alter table

```mysql
alter table table_name add index index_name (column, column)
alter table table_name add UNIQUE (column, column)
alter table table_name add primary key (column, column)
```

#### 2. create index（不能创建主键索引）

```mysql
create index index_name on table_name (column,column)
create unique index index_name on table_name (column, column)
```

#### 3.drop index

```mysql
drop index index_name on table_name
alter table table_name drop index index_name

-- 因为一个表只可能有一个PRIMARY KEY索引，因此不需要指定索引名
-- 如果没有创建PRIMARY KEY索引，但表具有一个或多个UNIQUE索引，则MySQL将删除第一个UNIQUE索引
alter table table_name drop primary key
```

### 注意事项

#### 1. 避免过度索引

比如性别可能只有两个值，建立索引不仅没有任何作用，还会影响到更新速度。这被称为过度索引。

#### 2. 推荐使用复合索引

```
select * from users where area=’beijing’ and age=22
```

如果我们是在area和age上分别创建单个索引的话，由于mysql查询每次只能使用一个索引，虽然这样已经相对不做索引时的全表扫描提高了很多效率，但是如果在area、age两列上创建复合索引的话将带来更高的效率。

如果我们创建了(area, age,salary)的复合索引，那么其实相当于创建了(area,age,salary)、(area,age)、(area)三个索引，这被称为最佳左前缀特性。

因此我们在创建复合索引时应该将最常用作限制条件的列放在最左边，依次递减。

#### 3. 索引不会包含有NULL值的列

只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为NULL。

#### 4. 避免左模糊查询

一般情况下不鼓励使用like操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引，而like “aaa%”可以使用索引。



## Nested Loop Join 嵌套循环连接

https://zhuanlan.zhihu.com/p/81398139

join有三种算法：

- Nested Loop Join
- Hash Join
- Sort Merge Join

MySQL目前只支持Nested Loop Join

>NLJ通过两层循环，用第一张表做Outter Loop，第二张表做Inner Loop。外循环的每条记录和内循环的记录作比较，符合条件就输出。

### 1. Simple Nested Loop Join

```
for (r in R) {
	for (s in S) {
		if (r satisfy condition s) {
			output <r, s>;
		}
	}
}
```

SNLJ让两张表做笛卡尔积，比较次数是R*S。

笛卡尔积：

(a, b) 笛卡尔积  (c, d)

结果：(a, c), (a, d), (b, c), (b, d)

实际上是排列组合的预期全部结果

应用：多表查询

1. 先确定要用到哪些表，将多个表通过笛卡尔积变成一个表
2. 然后去除不符合逻辑的数据（根据两个表的关系去除）
3. 最后当作一个虚拟表一样加上条件查询
### 2. Index Nested Loop Join

```
for (r in R) {
	for (si in SIndex) {
		if (r satisfy condition si) {
			output <r, si>;
		}
	}
}
```

INSJ在 SNLJ的基础上做了优化，**通过连接条件确定可用的索引**，Inner Loop中扫描索引而不去扫描全部数据，从而提高Inner Loop的效率。

缺点：如果扫描的索引是非聚簇索引，并且需要访问非索引的数据，会产生一个`回表`读取数据的操作，多了一次随机的I/O操作。

在MySQL5.6中，对INLJ的回表操作进行了优化，增加了Batched Key Access Join（批量索引访问的表关联方式）和Multi Range Read（mrr，多范围读取）特性，在join操作中缓存所需要的数据的rowid，再批量去获取其数据，把I/O从多次零散的操作优化为更少次数批量的操作，提高效率。

### 3. Block Nested Loop Join

   ```
   for (r in R) {
   	for (sbu in SBuffer) {
   		if (r satisfy condition sbu) {
   			output <r, sbu>;
   		}
   	}
   }
   ```

   BNLJ在SNLJ的基础上使用了join buffer，提前读取Inner Loop所需要的记录到buffer中，以提高Inner Loop的效率

## 为什么超过三张表禁止join

看完上面的嵌套循环连接的实现，这个问题就好回答了，每多一个join就多一层嵌套循环，代价太大了

## Inner Join、Left Join

- inner join  只返回两个表中联结字段相等的行。交集
- left join 返回包括左表中的所有记录、右表中联结字段相等的记录（不足的地方用NULL补充） 



## on与where

```mysql
SELECT * FROM classes;

id    name
1    一班
2    二班
3    三班
4    四班
```

```mysql
SELECT * FROM students;

id  class_id  name   gender
1    1        小明        M
2    1        小红        F
3    1        小军        M
4    1        小米        F
5    2        小白        F
6    2        小兵        M
7    2        小林        M
8    3        小新        F
9    3        小王        M
10    3        小丽        F
```

现在有两个需求：

1. 找出每个班级的名称及其对应的女同学数量

   ```mysql
   SELECT
   	c.NAME,
   	count( s.NAME ) AS num 
   FROM
   	classes c
   	LEFT JOIN students s ON s.class_id = c.id AND s.gender = 'F' 
   GROUP BY
   	c.NAME
   
   -- 查询结果
   name    num
   一班    2
   二班    1
   三班    2
   四班    0
   ```

   ```mysql
   SELECT
   	c.NAME,
   	count( s.NAME ) AS num 
   FROM
   	classes c
   	LEFT JOIN students s ON s.class_id = c.id 
   WHERE
   	s.gender = 'F' 
   GROUP BY
   	c.NAME
   	
   -- 查询结果 数据缺失，不符合预期
   name    num
   一班    2
   二班    1
   三班    2
   ```

2. 找出一班的同学总数

   ```mysql
   SELECT
   	c.NAME,
   	count( s.NAME ) AS num 
   FROM
   	classes c
   	LEFT JOIN students s ON s.class_id = c.id 
   WHERE
   	c.NAME = '一班' 
   GROUP BY
   	c.NAME
   	
   -- 查询结果
   name    num
   一班    4
   ```

   ```mysql
   SELECT
   	c.NAME,
   	count( s.NAME ) AS num 
   FROM
   	classes c
   	LEFT JOIN students s ON s.class_id = c.id AND c.NAME = '一班' 
   GROUP BY
   	c.NAME
   	
   -- 查询结果 多余数据，不符合预期
   name    num
   一班    4
   二班    0
   三班    0
   四班    0
   ```

分析：

- on：**先筛选再查询**。有了前面的嵌套循环连接的基础，不难看出，on用作inner join的判断条件，**只会对被关联表做限制**。因此第二个需求中，虽然on中限制了` c.NAME = '一班' `，但是主表classes表不受限制，还是会查询出全部数据。
- where：**先查询再根据条件筛选，对连接查询之后的临时表进行筛选**。因此第二个需求中，判断条件要放在where中

技巧：

- 如果想要主表的全部数据，则将条件放在on中。
  - 应用场景：有数据的汇总统计数据，没有数据的用默认值0表示

## 函数

### count()

>count() 是一个聚合函数。对于返回的结果集进行遍历，如果符合参数的判断条件，累计值就加一，否则不加，最后返回累计值。

#### count( [distinct] expression)

- 如果这个字段是`not null`，按行累加。
- 如果这个字段是允许为`null`的，遍历记录中的这个字段，取值，判断不为null，累加。

#### count(1) 

server层对于返回的每一行，放一个数字“1”“进去，因此判断是不可能为空的。

#### count(*)

所有行，直接按行累加

#### 结论：

**按照效率排序的话，count(字段)<count(主键id)<count(1)≈count(\*)，所以尽量使用count(\*)**

### DATA_FORMAT()

DATE_FORMAT() 函数用于以不同的格式显示日期/时间数据。

```mysql
DATE_FORMAT(date,format)

-- 常用
-- %Y 年份，4位，2022
-- %y 年份，两位，22

-- %M 月名
DATE_FORMAT(create_time ,'%Y-%m-%d')
```

应用：

按日统计用户优惠券的使用情况，形成折现趋势图

```mysql
SELECT
	count(*) DATA,
	DATE_FORMAT( t.use_time, '%Y-%m-%d' ) date 
FROM
	t_user_coupon t
	LEFT JOIN t_coupon c ON c.id = t.coupon_id 
WHERE
	c.shop_id = #{statistics.shopId}
	AND t.is_deleted = 0 
	AND t.use_time BETWEEN #{statistics.startTime} AND #{statistics.endTime}
GROUP BY
	date 
ORDER BY
	date DESC
```



## union

- union 

  并集，并且自动去重，不排序

- union all

  并集，不去重，不排序

- intersect

  交集，按照结果集的第一个列进行排序

- minus

  差集，按照结果集的第一个列进行排序

### 内排序

```mysql
select * from t_goods where price < 200 order by price asc
union
select * from t_goods where price > 100 order by price desc
```

上面SQL中，union前的排序为内层排序，不起作用。

如果先内层排序，然后外层再排序，明显内层排序是多余的，所有MySQL进行了优化，使得内层排序不起作用。

**如果想要内层排序生效，必须要使内层排序的结果能影响最终的结果。如:加上limit**



## 分组取最值

```mysql
SELECT
	*,
	MAX( create_time ) 
FROM
	sys_user
WHERE
	1 = 1 
GROUP BY
	user_id
```

## 折线图

```mysql
SELECT
	count(*) DATA,
	DATE_FORMAT( t.use_time, '%Y-%m-%d' ) date 
FROM
	t_user_coupon t
	LEFT JOIN t_coupon c ON c.id = t.coupon_id 
WHERE
	c.shop_id = #{statistics.shopId}
	AND t.is_deleted = 0 
	AND t.use_time BETWEEN #{statistics.startTime} AND #{statistics.endTime}
GROUP BY
	date 
ORDER BY
	date DESC
```

## 慢查询

### 定位

#### 开启慢查询日志

|        参数        |                             含义                             |
| :----------------: | :----------------------------------------------------------: |
|   slow_query_log   |                  是否启用慢查询。默认为OFF                   |
| slow_query_log_fle | 日志输出位置。默认为FILE，即保存为文件；若设置为TABLE，则保存到mysql.show_log表中 |
|     log_output     |                指定慢查询日志文件的路径和名字                |
|  long_query_time   |    执行时间超过该值才记录到慢查询日志，单位为秒，默认为10    |

##### 查看是否启用

```mysql
show variables like "%slow_query_log%"
```

##### 修改 配置文件

修改配置文件my.ini，在[mysqld]段落中加入如下参数

```ini
log_output='FILE,TABLE'
slow_query_log='ON'
long_query_time=0.001
```

**需要重启 MySQL 才可以生效，命令为 **

```shell
service mysqld restart
```

## 表连接 1对多的情况

```mysql
CREATE TABLE `t_user` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT 'id',
  `phone` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '手机号',
  `openid` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '小程序openid',
  `nickname` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '昵称',
  `avatar_url` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '头像url',
  `birthday` date DEFAULT NULL COMMENT '用户生日',
  `sex` tinyint(1) NOT NULL DEFAULT '0' COMMENT '0未知 1 男 2 女',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '更新时间',
  `is_deleted` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否删除',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=25 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='用户表';
```

```mysql
CREATE TABLE `t_shop_vip` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `shop_id` bigint NOT NULL COMMENT '门店id',
  `user_id` bigint NOT NULL COMMENT '用户id',
  `real_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL COMMENT '真实姓名',
  `balance` decimal(10,2) NOT NULL COMMENT '余额',
  `points` int NOT NULL COMMENT '积分',
  `note` varchar(60) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL COMMENT '备注',
  `create_time` datetime NOT NULL COMMENT '创建时间',
  `update_time` datetime NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`) USING BTREE,
  KEY `idx_user_id` (`user_id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=20 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='会员用户表';
```

两张表，一个微信账号就是user，但是一个user可以在不同shop入会成为shop_vip。user和shop_vip的关系属于一对多。

```mysql
SELECT
	o.order_no,
	o.pay_price,
	o.type orderType,
	o.STATUS,
	o.create_time,
	u.avatar_url,
	sv.real_name 
FROM
	t_order o
	LEFT JOIN t_user u ON u.id = o.user_id
	// 这里存在情况：一个user可以在t_shop_vip中有多条记录
	LEFT JOIN t_shop_vip sv ON sv.user_id = u.id 
WHERE
	o.shop_id = 1
ORDER BY
	o.create_time DESC
```

```mysql
O20220517110713190	20.00	4	5	2022-05-17 18:51:41	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O20220517110713190	20.00	4	5	2022-05-17 18:51:41	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O20220517110713190	20.00	4	5	2022-05-17 18:51:41	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员
O2022051710441391	15.00	3	1	2022-05-17 12:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O2022051710441391	15.00	3	1	2022-05-17 12:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O2022051710441391	15.00	3	1	2022-05-17 12:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员
O2022051710391270	15.00	3	0	2022-05-17 10:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O2022051710391270	15.00	3	0	2022-05-17 10:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O2022051710391270	15.00	3	0	2022-05-17 10:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员
```

预期效果

```mysql
O20220517110713190	20.00	4	5	2022-05-17 18:51:41	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O2022051710441391	15.00	3	1	2022-05-17 12:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
O2022051710391270	15.00	3	0	2022-05-17 10:38:13	https://img.yukicomic.com/ciyo/default_avatarUrl.png	会员1
```

### 方法一：先关联再筛选

```mysql
SELECT
	o.order_no,
	o.pay_price,
	o.type orderType,
	o.STATUS,
	o.create_time,
	u.avatar_url,
	sv.real_name 
FROM
	t_order o
	LEFT JOIN t_user u ON u.id = o.user_id
	LEFT JOIN t_shop_vip sv ON sv.user_id = u.id 
WHERE
	o.shop_id = 1 
	AND sv.shop_id = 1 
ORDER BY
	o.create_time DESC
```

![image-20220527095902240](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220527095902240.png)

### 方法二：关联时增加on条件  实现一对一

```mysql
SELECT
	o.order_no,
	o.pay_price,
	o.type orderType,
	o.STATUS,
	o.create_time,
	u.avatar_url,
	sv.real_name 
FROM
	t_order o
	LEFT JOIN t_user u ON u.id = o.user_id
	LEFT JOIN t_shop_vip sv ON sv.user_id = u.id
  AND sv.shop_id = 1
WHERE
	o.shop_id = 1 
-- 	AND sv.shop_id = 1 
ORDER BY
	o.create_time DESC
```

![image-20220527095740593](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220527095740593.png)

