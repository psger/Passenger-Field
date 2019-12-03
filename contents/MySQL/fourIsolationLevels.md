### 隔离级别
数据库事务有不同的隔离级别，不同的隔离级别对锁的使用是不同的，锁的应用最终导致不同事务的隔离级别。
实现隔离级别的方式就是加锁
### 隔离级别的分类
1. 读未提交 Read Uncommitted（在本次事务中可以读到其他事务中没有提交的数据-脏数据）
2. 读已提交 Read Committed （只能读到其他事务提交过的数据。如果在当前事务中，其他事务有提交，则两次读取结果不同）
3. 可重复读 Repeatable Read （默认，保证了事务中每次读取结果都相同，而不管其他事物是否已经提交。会出现幻读）
4. 序列化 Serializable （隔离级别中最严格的，开启一个serializable事务，那么其他事务对数据表的写操作都会被挂起）

### 实验准备
1. 首先创建一个表account
```sql
CREATE TABLE account(
	id INT NOT NULL AUTO_INCREMENT,
	ACCOUNT FLOAT NOT NULL,
	PRIMARY KEY(id)
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
然后插入数据
![](https://www.psgers.com/uploads/article/20180701/d5c03dfe11ee5e59f5507d18965d0911.png)
为了说明问题，我们打开两个控制台分别进行登录来模拟两个用户（暂且成为用户A和用户B吧），并设置当前MySQL会话的事务隔离级别。

2. 设置事务隔离级别
设置innodb的事务级别方法是：set 作用域 transaction isolation level 事务隔离级别
```sql
SET [SESSION | GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}
```
```
mysql> set global transaction isolation level read committed; //全局的
mysql> set session transaction isolation level read committed; //当前会话
```

### 实验过程
#### 1.read uncommitted（读取未提交数据）
A用户
```
set session transaction isolation level read uncommitted;
start transaction;
select * from account;
```
![](https://www.psgers.com/uploads/article/20180701/d5c03dfe11ee5e59f5507d18965d0911.png)
B用户
```
set session transaction isolation level read uncommitted;
start transaction;
update account set account=account+200 where id = 1;
```
A用户查询
![](https://www.psgers.com/uploads/article/20180701/95c9294cb8ffe1379ff8acc5e3f744c7.png)
##### 结论一：我们将事务的隔离级别设置为读未提交，所以A用户能够读到B用户没有提交的数据
存在的问题：那么这么做有什么问题吗？
那就是我们在一个事务中可以随随便便读取到其他事务未提交的数据，这还是比较麻烦的，我们叫脏读。实际上我们的数据改变了吗？答案是否定的，因为只有事务commit后才会更新到数据库。

#### 2.read committed（可以读取其他事务提交的数据）---大多数数据库默认的隔离级别
将B用户的隔离级别设置为read committed
```
set session transaction isolation level read committed
```
A用户
```
start transaction;
update account set account = account-200 where id = 1;
select * from account;
```
![](https://www.psgers.com/uploads/article/20180701/93da0bf01a3f2b2cbfbed89f646a39f2.png)
上图可知A中能查询到数据的变化

B中
![](https://www.psgers.com/uploads/article/20180701/c49e917718eca3c3202d12f541dea76b.png)
没有查询到数据的变化
在A中commit之后再在B中查询
![](https://www.psgers.com/uploads/article/20180701/93da0bf01a3f2b2cbfbed89f646a39f2.png)

##### 结论二：我们将当前会话的隔离级别设置为read committed的时候，当前会话只能读取到其他事务提交的数据，未提交的数据读不到。
存在的问题：那就是我们在会话B同一个事务中，读取到两次不同的结果。这就造成了不可重复读，就是两次读取的结果不同。这种现象叫不可重复读。

#### 3.repeatable read（可重读）---MySQL默认的隔离级别
设置B中的隔离级别为repeatable read
```
set session transaction isolation level repeatable read;
start transaction;
select * from account;
```
![](https://www.psgers.com/uploads/article/20180701/7a149fbbfaa1d0140d9dea6578433f67.png)
在A中添加一条数据
```
insert into account(id,account) value(3,1000);
commit;
```
![](https://www.psgers.com/uploads/article/20180701/df6bacc2ab65d6f08d0833558c788fcf.png)
在B中再查询：
![](https://www.psgers.com/uploads/article/20180701/7a149fbbfaa1d0140d9dea6578433f67.png)
用户B在他所在的会话中想插入一条新数据id=3，value=1000。来我们操作下：
```
insert into account(id,account) value(3,5000);
```
![](https://www.psgers.com/uploads/article/20180701/0172097808d97b8ec2d36e3e578031fe.png)
发现插入不进去
##### 结论三：当我们将当前会话的隔离级别设置为repeatable read的时候，当前会话可以重复读，就是每次读取的结果集都相同，而不管其他事务有没有提交。
出现的问题：
一个事务中读取的数据一致（可重复读），数据已经发生改变，但是我还是要保持一致。但是，出现了用户B面对的问题，这种现象叫幻读。

#### 4.serializable（串行化）
在B中先开启serializable事务
```
set session transaction isolation level serializable;
start transaction;
select * from account;
```
然后在A中写数据，会出现超时，如果这时B commmit,那么A中会执行成功
#### 结论四：当我们将当前会话的隔离级别设置为serializable的时候，其他会话对该表的写操作将被挂起。可以看到，这是隔离级别中最严格的，但是这样做势必对性能造成影响。所以在实际的选用上，我们要根据当前具体的情况选用合适的。

### 总结:
1. 读未提交:别人修改数据的事务尚未提交,在我的事务中也能读到.
2. 读已提交:别人修改数据的事务已经提交,在我的事务中才能读到.
3. 可重复读:别人修改数据的事务已经提交,在我的事务中也读不到.
4. 串行:我的事务尚未提交,别人就别想改数据.
这四种隔离级别,从上到下,并行能力依次降低,安全性一次提高.