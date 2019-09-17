## 目标

生成全局唯一整数

不使用GUID，是因为GUID比较大，而且不利于搜索

## 方案

若使用MySQL的自增ID，但保证不了分库之后的唯一性。既然这样就把ID生成放在中心数据库（只生成ID，不存其他表项）。

利用MySQL的`REPLACE INTO`。`REPLACE`和`INSERT`类似，区别在于在PRIMARY KEY或者UNIQUE索引冲突时，新的行会替换老的行。

## 表结构

多个表，32位ID用Tickets32，64位ID用Tickets64。根据stub区分用途，因为stub是唯一的，只会保持一行，同时ID也会自增。

```sql
CREATE TABLE `Tickets64` (
    `id` bigint(20) unsigned NOT NULL auto_increment,
    `stub` char(1) NOT NULL default '',
    PRIMARY KEY  (`id`),
    UNIQUE KEY `stub` (`stub`)
) ENGINE=MyISAM
```

```sql
SELECT * from Tickets64

+-------------------+------+
| id                | stub |
+-------------------+------+
| 72157623227190423 |    a |
+-------------------+------+
```

获取新的ID

```sql
REPLACE INTO Tickets64 (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

为了避免单点故障，开多个ticket server，然后RR取ID；或者可以考虑分表，放在不同的服务器提高并发。

```
TicketServer1:
auto-increment-increment = 2
auto-increment-offset = 1

TicketServer2:
auto-increment-increment = 2
auto-increment-offset = 2
```

## Reference

+ [Ticket Servers: Distributed Unique Primary Keys on the Cheap](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
