# go操作mysql

## 连接

golang中，`database/sql`包提供了保证sql或类sql数据库的泛用接口，并不提供具体的数据库驱动，相当于定义了一个通用规范，使用`database/sql`包时必须注入一个数据库驱动（其必须实现sql包定义的接口规范）

常用的数据库都有现成的连接库实现：[如myql驱动](https://github.com/go-sql-driver/mysql)

###　下载依赖

```
$ go get -u github.com/go-sql-driver/mysql
```



### 使用mysql驱动



Open函数签名：

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

其中：

- driveName表示数据库类型，mysql，pgsql这种
- dataSourceName表示数据库连接字符串，ip端口用户名密码这些
- 返回一个DB类型的指针类型，和可能的error

连接示例：

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// DSN:data source name
	dsn := "root:123456@tcp(192.168.80.100:3306)/mysql"
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}
	defer db.Close()
	fmt.Println("connect mysql is successful")
}
$ ./mysql.exe
connect mysql is successful
```



### 初始化连接

上例中，使用sql.Open只是校验了参数的准确与否，并不实际检测于mysql的连接是否正常，若要检测，需要用db对象的Ping()方法

**返回的DB对象，可以安全被多个goroutine并发使用，并维护自己的空闲连接池，open函数仅仅应该被调用一次，很少需要关闭DB对象**

e.g.

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

// 定义一个全局的DB对象
var db *sql.DB

func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/db1?charset=utf8mb4&parseTime=True"
    // 并不尝试与数据库建立连接
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

    // 测试与数据库的连接
	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")
}


$ ./mysql.exe
init db failed Error 1130: Host '192.168.80.1' is not allowed to connect to this MariaDB server


因为mysql默认没有放行程序所在ip，所以不通
放行
MariaDB [(none)]> grant all on *.* to root@'192.168.80.%' identified by '123456';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.01 sec)


再运行
$ ./mysql.exe
init db success!
```

*sql.DB是表示连接的数据库对象，结构体，保存了数据库连接相关的信息，内部维护了一个具有0到多个底层连接的连接池，可以被安全的被goroutine并发使用

定义：

```go
type DB struct {
	// Atomic access only. At top of struct to prevent mis-alignment
	// on 32-bit platforms. Of type time.Duration.
	waitDuration int64 // Total time waited for new connections.

	connector driver.Connector
	// numClosed is an atomic counter which represents a total number of
	// closed connections. Stmt.openStmt checks it before cleaning closed
	// connections in Stmt.css.
	numClosed uint64

	mu           sync.Mutex // protects following fields
	freeConn     []*driverConn
	connRequests map[uint64]chan connRequest
	nextRequest  uint64 // Next key to use in connRequests.
	numOpen      int    // number of opened and pending open connections
	// Used to signal the need for new connections
	// a goroutine running connectionOpener() reads on this chan and
	// maybeOpenNewConnections sends on the chan (one send per needed connection)
	// It is closed during db.Close(). The close tells the connectionOpener
	// goroutine to exit.
	openerCh          chan struct{}
	closed            bool
	dep               map[finalCloser]depSet
	lastPut           map[*driverConn]string // stacktrace of last conn's put; debug only
	maxIdle           int                    // zero means defaultMaxIdleConns; negative means 0
	maxOpen           int                    // <= 0 means unlimited
	maxLifetime       time.Duration          // maximum amount of time a connection may be reused
	cleanerCh         chan struct{}
	waitCount         int64 // Total number of connections waited for.
	maxIdleClosed     int64 // Total number of connections closed due to idle.
	maxLifetimeClosed int64 // Total number of connections closed due to max free limit.

	stop func() // stop cancels the connection opener and the session resetter.
}
```



### SetMaxOpenConns

```go
func (db *DB) SetMaxOpenConns(n int)
```

设置最大能打开的连接数目，若n大于0，小于最大空闲的连接数，则最大空闲连接数会减少到匹配最大开启连接的限制

### SetMaxIdleConns

```go
func (db *DB) SetMaxIdleConns(n int)
```

设置最大空闲连接数

## CRUD

### 建库建表

1、建立测试库

```mysql
MariaDB [sql_test]> create database sql_test;

```



2、切换到测试库

```mysql
MariaDB [sql_test]> use sql_test;

```





3、创建测试表

```mysql
MariaDB [sql_test]> CREATE TABLE `user` (
    ->     `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
    ->     `name` VARCHAR(20) DEFAULT '',
    ->     `age` INT(11) DEFAULT '0',
    ->     PRIMARY KEY(`id`)
    -> )ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
Query OK, 0 rows affected (0.01 sec)

```



### 查询数据

0、向user表插入几条数据

```mysql
MariaDB [sql_test]> insert user( name, age) values("wang", 24);
Query OK, 1 row affected (0.00 sec)

MariaDB [sql_test]> insert user( name, age) values("li", 22);
Query OK, 1 row affected (0.00 sec)

MariaDB [sql_test]> insert user( name, age) values("zhangsan", 18);
Query OK, 1 row affected (0.00 sec)

MariaDB [sql_test]> select * from user;
+----+----------+------+
| id | name     | age  |
+----+----------+------+
|  1 | wang     |   24 |
|  2 | li       |   22 |
|  3 | zhangsan |   18 |
+----+----------+------+

```



1、定义对user表对应的结构体，用于存储一条数据条目

```go
全局定义


type user struct {
	id   int
	name string
	age  int
}

```



2、单行查询

单行查询函数签名：

```go
func (db *DB) QueryRow(query stirng, args ...interface{}) *Row
```



```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func queryRow() {

	var u1 user // 声明一个变量u1
	sqlStr := `select id ,name, age  from sql_test.user where id = ?`
	// QueryRow传入一个包含占位符的字符串，占位符用？表示
	// QueryRow得到一个*Row对象，该对象的Scan方法可以将查的数据存放到其参数的变量中
    // QueryRow之后，一定调用Scan方法，来释放数据库连接
	err := db.QueryRow(sqlStr, 1).Scan(&u1.id, &u1.name, &u1.age)
	if err != nil {
		fmt.Println("query and scan failed")
		return
	}
	fmt.Println(u1.id, u1.name, u1.age)
}
func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/db1?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	queryRow()
}

$ ./mysql.exe
init db success!
1 wang 24

```



3、多行查询

多行查询函数Query签名

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

e.g.

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func queryMutilRow() {
	var u1 user

	sqlStr := `select id , name, age from sql_test.user where id > ?`

	rows, err := db.Query(sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err : %v\n", err)
		return
	}

	// 注册一个关闭连接函数
	defer rows.Close()

    // 遍历多行
	for rows.Next() {
		err = rows.Scan(&u1.id, &u1.name, &u1.age)
		if err != nil {
			fmt.Println("scan failed")
			continue
		}
		fmt.Println(u1.id, u1.name, u1.age)

	}
}

func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/db1?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	// queryRow()
	queryMutilRow()
}

$ ./mysql.exe
init db success!
1 wang 24
2 li 22
3 zhangsan 18
```



### 插入数据

插入、更新、删除操作都用Exec方法

```go
func (db *DB) Exec(query string, args ...interface{}) (Result , error)
```

**返回的Result是sql命令的执行结果的提示信息，args是 query字符串的占位参数**

插入demo

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/db1?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func insertOne() {
	sqlStr := `insert into sql_test.user(name, age) values(?, ?)`
	res, err := db.Exec(sqlStr, "haha", 18)
	if err != nil {
		fmt.Println("insert failed", err)
		return
	}

	id, err := res.LastInsertId()
	if err != nil {
		fmt.Println("get last id failed", err)
		return
	}

	fmt.Println("insert id is : ", id)
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	insertOne()
}

```

运行结果

```go
$ ./mysql.exe
init db success!
insert id is :  4

MariaDB [sql_test]> select * from user 
    -> ;
+----+----------+------+
| id | name     | age  |
+----+----------+------+
|  1 | wang     |   24 |
|  2 | li       |   22 |
|  3 | zhangsan |   18 |
|  4 | haha     |   18 |
+----+----------+------+
4 rows in set (0.00 sec)

```



### 更新数据

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/db1?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func updataDemo() {
	sqlStr := `update sql_test.user set age = ? where id > ?`
	res, err := db.Exec(sqlStr, 100, 1)
	if err != nil {
		fmt.Println("update failed", err)
		return
	}

	num, err := res.RowsAffected()
	if err != nil {
		fmt.Println("row affect  failed", err)
		return
	}

	fmt.Printf("there was %d line was changed\n", num)
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	updataDemo()
}

```

运行结果

```go
$ ./mysql.exe
init db success!
there was 3 line was changed


MariaDB [sql_test]> select * from user;
+----+----------+------+
| id | name     | age  |
+----+----------+------+
|  1 | wang     |   24 |
|  2 | li       |  100 |
|  3 | zhangsan |  100 |
|  4 | haha     |  100 |
+----+----------+------+
4 rows in set (0.00 sec)


```



### 删除数据

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/db1?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func deleteDemo() {
	sqlStr := `delete  from sql_test.user where id = ?`
	res, err := db.Exec(sqlStr, 1)
	if err != nil {
		fmt.Println("delete failed", err)
		return
	}

	num, err := res.RowsAffected()
	if err != nil {
		fmt.Println(" get delelte operation affect  num failed", err)
		return
	}

	fmt.Printf("there was %d line was deleted \n", num)
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	deleteDemo()
}

```



运行结果：

```
$ ./mysql.exe
init db success!
there was 1 line was deleted

MariaDB [sql_test]> select * from user;
+----+----------+------+
| id | name     | age  |
+----+----------+------+
|  2 | li       |  100 |
|  3 | zhangsan |  100 |
|  4 | haha     |  100 |
+----+----------+------+
3 rows in set (0.00 sec)

```

### 总结：

查询使用db对象的QueryRow（单行）和Query（多行）方法，而插入、更新、删除都是用db对象的Exec方法；得到的对象有2个方法，分别是`LastInsertId()`和`RowsAffected()`

## mysql预处理

### 什么是预处理

**普通sql执行过程**

1. go程序把查询sql语法，发送给后端数据库，
2. 数据库对sql进行编译，然后执行
3. 将执行后的结果，返回给go程序

**预处理sql执行过程**

1. 把sql分为2部分，命令部分和数据部分
2. 将命令部分（常用且重复）发给数据库进行编译
3. 然后把每次要执行的sql的数据部分（重复频率不高）发给数据库编译
4. 数据库结合命令、数据编译部分的结果，执行命令
5. 返回结果给go程序



### 为什么要预处理

**好处：**

1. 将重复用的命令部分一次编译，多次使用，减少了部分sql编译时间
2. 避免sql注入问题

### go实现mysql预处理

`database/sql`中使用`Prepare()`方法实现预处理操作

```go
func (db *DB) Prepare(query string) (*Stmt , error)
```

Prepare方法，会先将sql发到数据库，然后返回一个准备好的状态，即*Stmt对象，用于之后的命令，返回值可以同时执行多个查询命令

**查询预处理示例代码**：

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func prepareQueryDemo() {

	var u user

	// 先把包含占位符的sql语句发到数据库
	sqlStr := `select id, name , age from user where id > ?`
	stat, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Println("prepare failed : ", err)
		return
	}
	defer stat.Close()
	// 用得到的stat的Query方法查询，把要查询的sql的数据部分传入
	rows, err := stat.Query(1)
	if err != nil {
		fmt.Println("stat query failed, : ", err)
		return
	}

	defer rows.Close()
	// 得到的是row，利用Next（）方法遍历
	for rows.Next() {
		err = rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Println("scan failed, :", err)
			continue
		}

		fmt.Println(u.id, u.name, u.age)
	}

}
func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/sql_test?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	prepareQueryDemo()
}

```



运行结果：

```
$ ./mysql.exe
init db success!
2 li 100
3 zhangsan 100
4 haha 100

```

**插入预处理示例：**

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func prepareInsertDemo() {

	// 先把包含占位符的sql语句发到数据库
	sqlStr := `insert user(name, age) values(?, ?)`
	stat, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Println("prepare failed : ", err)
		return
	}
	defer stat.Close()

	_, err = stat.Exec("heihei", 18)
	if err != nil {
		fmt.Println("stat exec first failed, ", err)
		return
	}

    // 执行多次
	_, err = stat.Exec("haohaohao", 78)
	if err != nil {
		fmt.Println("stat exec second failed, ", err)
		return
	}

	fmt.Println("insert two sucess!")

}
func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/sql_test?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	prepareInsertDemo()
}


```

运行结果：

```
$ ./mysql.exe
init db success!
insert two sucess!

MariaDB [sql_test]> select * from user;
+----+-----------+------+
| id | name      | age  |
+----+-----------+------+
|  2 | li        |  100 |
|  3 | zhangsan  |  100 |
|  4 | haha      |  100 |
|  5 | heihei    |   18 |
|  6 | haohaohao |   78 |
+----+-----------+------+
5 rows in set (0.00 sec)

```

**更新、删除和更新类似**

**总结：**

1. 和普通的sql查询不同的步骤是：
2. 先利用db.Prepare()把要执行的sql语句（包含占位符）发送给服务端
3. 得到的*Stmt对象，执行Query或Exec方法，再把与占位符对应的部分传入，数据库端得以执行完整的sql，然后返回结果
4. 得到*Stmt类型的变量时，注意defer 对象变量名.Close()注册一个关闭操作
5. 得到的*Stmt对象，可以执行多次命令

### sql注入

任何时候，都不应该自行拼接sql语句，**下为一个自行拼接sql的示例：实现一个根据name字段查询user表的函数如下**

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

func sqlInjectDemo(name string) {
	sqlStr := fmt.Sprintf("select id, name, age from user where name = '%s'", name)
	fmt.Println(sqlStr)

	var u user
	rows, err := db.Query(sqlStr)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer rows.Close()

	for rows.Next() {
		err = rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Println("scan failed", err)
			continue
		}
		fmt.Printf("user: %#v \n", u)
	}

}

func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/sql_test?charset=utf8mb4&parseTime=True"
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	sqlInjectDemo("xxx ' or '1=1")

}

```

**用户输入正常时**

```
$ ./mysql.exe
init db success!
select id, name, age from user where name = 'wang'
user: main.user{id:1, name:"wang", age:18}
```

**此时用户输入以下字符串作为查询条件，都会导致sql注入，引发安全问题**

```go



sqlInjectDemo("xxx ' or '1=1")
// 同样的还有：
sqlInjectDemo("xxx' union select * from user #")
sqlInjectDemo("xxx' and (select count(*) from user) < 10 #")


$ ./mysql.exe
init db success!
select id, name, age from user where name = 'xxx ' or '1=1'
user: main.user{id:1, name:"wang", age:18}
user: main.user{id:2, name:"li", age:22}
user: main.user{id:3, name:"gu", age:24}
```



**不同的数据库中，sql语句中，使用的占位符不尽相同**

| 数据库     | 占位符语法 |
| ---------- | ---------- |
| mysql      | ？         |
| postgreSQL | $1 $2      |
| sqllite    | ? $1       |
| oracle     | :name      |



## go实现mysql事务

### 什么是事务

一组sql的操作，要么全部成功，要么失败回滚，该组sql组成的即是事务，典型的有银行的转账操作；



### 事务的ACID

事务通常需要满足4个条件，ACID，原子性atomicity，一致性Consistency，隔离性Isolation，持久性Durability

|  条件  |                             解释                             |
| :----: | :----------------------------------------------------------: |
| 原子性 | 一个事务（transaction）中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。 |
| 一致性 | 在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性地完成预定的工作。 |
| 隔离性 | 数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括读未提交（Read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（Serializable）。 |
| 持久性 | 事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。 |

### 事务相关方法

Golang中用以下三个方法实现mysql中的事物操作

**开始事务**

```
func (db *DB) Begin() (*Tx, error)
```

**提交事务**

```
func (tx *TX) Commit() error
```

**回滚事务**

```
func (tx *TX) Rollback() error
```



### 事务示例

下面为一个简单的事务操作，其中操作可以保证2次更新要么都成功，要么都失败，不存在中间状态

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

type user struct {
	id   int
	name string
	age  int
}

/*
首先利用db.Begin()方法开启一个事务，
	得到一个事务对象tx，和一个error类型err，判断err是否为空，不是这返回
		判断tx是否为空，不是就返回

	正常得到开启事务后：
	利用事务的Exec（）方法执行一条更新语句，
		判断得到的err，不为空则返回
		判断得到影响的行数变量，不为空则返回

	再执行一条更新语句，
		判断得到的err，不为空则返回
		判断得到影响的行数变量，不为空则返回

	然后利用与操作判断2次更新的行数是否都为1，
		都为1是，进行tx的提交事务操作
		否则，进行tx的回滚操作


*/

func transactionDemo() {

	tx, err := db.Begin()
	// 得到一个tx对象，和可能的err
	if err != nil {
		if tx != nil {
			tx.Rollback()
		}
		fmt.Println("事务开启失败:", err)
		return
	}

	// 给用户id为1的用户年龄加100
	sqlStr1 := `update user set age=age + 100 where id = ?`
	ret1, err := tx.Exec(sqlStr1, 1)
	if err != nil {
		fmt.Println("执行sql1失败，", err)
		return
	}
	affectRow1, err := ret1.RowsAffected()
	if err != nil {
		fmt.Println("得到sql1执行后受影响的行数失败", err)
		return
	}

	// 给用户id为2的用户年龄减去100
	sqlStr2 := `update user set age=age - 100 where id = ?`
	ret2, err := tx.Exec(sqlStr2, 2)
	if err != nil {
		fmt.Println("执行sql2失败，", err)
		return
	}
	affectRow2, err := ret2.RowsAffected()
	if err != nil {
		fmt.Println("得到sql2执行后受影响的行数失败", err)
		return
	}

	// 判断2个语句是否都执行成功，若是，则进行事务的提交
	if affectRow1 == 1 && affectRow2 == 1 {
		fmt.Println("事务将要进行提交")
		tx.Commit()
	} else {
		fmt.Println("事务将进行回滚")
		tx.Rollback()
	}

}

func initDB() (err error) {
	dsn := "root:123456@tcp(192.168.80.100:3306)/sql_test?charset=utf8mb4&parseTime=True"
	// 并不尝试与数据库建立连接
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err

	}

	// 测试与数据库的连接
	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {

	err := initDB()
	if err != nil {
		fmt.Println("init db failed", err)
		return
	}
	fmt.Println("init db success!")

	transactionDemo()
}

```

事务提交的结果

```
MariaDB [sql_test]> select * from user;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | wang |   18 |
|  2 | li   |   22 |
|  3 | gu   |   24 |
+----+------+------+
3 rows in set (0.00 sec)

MariaDB [sql_test]> select * from user;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | wang |  118 |
|  2 | li   |  -78 |
|  3 | gu   |   24 |
+----+------+------+
3 rows in set (0.00 sec)

```

事务执行失败，回滚的结果

```
// 给用户id为25的用户年龄加100
因为找不到id为25的用户，所以这句执行不成功，所以下面的if判断，就会执行事务回滚的逻辑，
MariaDB [sql_test]> update user set age=age + 100 where id = 25;
Query OK, 0 rows affected (0.00 sec)
Rows matched: 0  Changed: 0  Warnings: 0



	sqlStr1 := `update user set age=age + 100 where id = ?`
	ret1, err := tx.Exec(sqlStr1, 25)
	if err != nil {
		fmt.Println("执行sql1失败，", err)
		return
	}
	
数据表，也不会有任何变化
MariaDB [sql_test]> select * from user;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | wang |   18 |
|  2 | li   |   22 |
|  3 | gu   |   24 |
+----+------+------+
3 rows in set (0.00 sec)

```



# 练习

1. 结合`net/http`和`database/sql`实现一个使用MySQL存储用户信息的注册及登陆的简易web程序。

暂略