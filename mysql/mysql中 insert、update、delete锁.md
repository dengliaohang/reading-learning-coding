# mysql中 insert、update、delete锁
### 对于表的锁的探索
#### 开多个客户端界面
```
DROP TABLE IF EXISTS `m_user`;
CREATE TABLE `m_user` (
  `i_id` int(10) unsigned zerofill NOT NULL AUTO_INCREMENT,
  `i_name` varchar(255) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  `is_delete` varchar(1) DEFAULT NULL,
  `i_type` varchar(5) DEFAULT NULL,
  PRIMARY KEY (`i_id`),
  UNIQUE KEY `index_i_id` (`i_id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

```
#### 测试数据:
```

INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000001', 'ajason', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '1');
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000002', 'tom', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '2');
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000003', 'jane', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '3');
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000004', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '4');

```
##### 前提 i_type 普通字段，无索引

### 若字段条件(where 条件)的值在表中并无记录，是不会触发锁的 (即影响行数0)



### alter table 操作，不需要使用事务 (也没有作用) , 遇到表中未提交的事务会被阻塞



#### 事务开启：begin ;
#### 事务回归：rollback ;
#### 事务提交：commit ;

### update 锁
```
事务A 中执行update 时( 未提交commit )，若使用主键查询，锁的是行记录
1. update m_user set i_name = 'ajason' where i_id = 1;
2. update m_user set i_name = 'ajason' where i_id in (1  , 2);

事务B 中
使用 insert delete update 对 1.语句中 i_id = 1 行进行操作时，会发生阻塞
使用 insert delete update 对 2.语句中 i_id = 1 与 2 的行进行操作时，会发生阻塞
```

#### 若用非主键查询，锁的是整表，这时候事务还未提交时，插入(update)删除(delete)操作都阻塞等待
后果：容易死锁
如：
```
事务A 中执行( 未提交 )：update m_user set i_name = 'ajason' where i_type in ( 1 )
其他事务中对表中数据进行的 insert , update , delete 操作都会被阻塞
```
同理 ：


### delete 锁
#### 事务中执行非主键查询锁表，主键查询锁行
```
delete from  m_user  where i_type in ( 1 )
delete from  m_user where i_id = 1;
```

### 当表中还有事务 未提交
#### alter table 对表进行修改会造成死锁


## 这时候将 i_type 改成 unique 的 btree 索引时
```
index_i_type	i_type	Unique
```



### 操作(update,delete同)
```
事务A 中执行以上操作并且未提交:
update m_user set i_name = 'ajason' where i_type in ( 1 )
update m_user set i_name = 'ajason' where i_type = 1 
另一个事务B 中执行
update m_user set i_name = 'ajason' where i_type = 4 (或者别的值，主要测试间隙锁)
这时候事务B阻塞
```



### insert锁
#### 当事务A执行如下语句时(i_id 为 5 ,原来数据库没有的字段)

```
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000005', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '5');
```

#### 事务A未提交
##### 1.其他事务对i_id 为 5 的操作无法继续(阻塞等待)
比如执行(同理delete)
```
行锁：update m_user set i_name = 'ajason' where i_id = 5;
表锁：update m_user set i_name = 'ajason' where i_type = 1  (不论i_type是多少)
```
##### 2.其他事务对i_id 不为 5 的操作不会阻塞
比如执行(同理delete)

```
行锁：update m_user set i_name = 'ajason' where i_id = 6;
```

##### 或者执行 insert 操作 分3种情况分析：
1.主键与unique索引都相同
```
事务B 执行:
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000005', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '5');
两个事务都阻塞，晚提交的事务报错
```

2.主键相同，索引不同

```
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000005', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '6');
两个事务都阻塞，晚提交的事务报错
```

3.主键不同，索引相同
```
INSERT INTO `dragsun_db`.`m_user` (`i_id`, `i_name`, `create_time`, `update_time`, `is_delete`, `i_type`) VALUES ('0000000006', 'jenny', '2017-06-13 09:03:08', '2017-07-24 09:03:11', '1', '5');
两个事务都阻塞，晚提交的事务报错
```

4.主键和索引都不同
互不影响



### 结论：
#### 说明行锁通过主键，唯一(unique)索引无法锁行(与普通字段一样)，只会锁表


### 锁表：
```
使用 insert , update , delete 语句对表操作时，都会阻塞
锁行：
对该行进行操作时才会阻塞
```

### 锁表和锁行时：
#### 进行普通 select 查询并不会阻塞
### select 锁(排他锁和共享锁)
```
非：
select * from  m_user (条件) for update
select * from  m_user (条件) lock in share mode
加条件的时候就要区分行锁表锁的select 查询了
同理，也是分为主键锁和unique锁
主键条件锁行
其他条件锁表
```

#### 数据库实现了超时锁释放
#### 数据库锁超时报错：
```
[Err] 1205 - Lock wait timeout exceeded; try restarting transaction
```

### 操作:
#### 当遇到alter table 对表操作时，直接无响应 或者死锁
```
select * from information_schema.INNODB_TRX
select * from information_schema.INNODB_LOCKS
select * from information_schema.INNODB_LOCK_WAITS
show PROCESSLIST

杀掉进程
kill 45(processlist 的 id )
接着提交回滚未完成的事务，然后继续对表操作
```