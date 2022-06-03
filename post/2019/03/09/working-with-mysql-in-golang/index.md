# 在Golang中对MySQL进行操作


Golang官方并没有提供数据库驱动，但通过`database/sql/driver`包来提供了实现驱动的标准接口。可以在Github上找到很多开源的驱动。
其中`go-sql-driver/mysql`是一个比较推荐的驱动，其完全支持`database/sql`接口。
使用这个驱动, 在项目里import进：

```go
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)
```
在正式使用`database/sql`包之前，首先得明白`sql.DB`并不代表一个数据库连接，它并不会与数据库建立任何连接，也不会验证参数的合法性，要想知道DSN的合法性，需使用sql.DB实例（比如db）`db.Ping()` 方法， 如下：
```go
err = db.Ping()
if err != nil {
  // 错误处理
}
```
使用`sql.Open()`方法即可获得一个`sql.DB`实例。需要注意的是，`sql.DB`的设计就是用来作为长连接使用的，不应该在项目里频繁的进行`Open() `与`Close()`，提倡的做法是声明一个全局的sql.DB实例, 将其复用起来。即只`Open()`一次，使用直到程序结束任务。

拿到`sql.DB`实例之后，就可以对数据库进行操作了。
在操作数据库时，推荐做法是使用`db.Prepare()`对`SQL`语句进行预编译，这样具有较高的安全性，可在一定程度上避免诸如`SQL注入`这样的攻击手段。

一些示例：
```go
/*
  查询操作
*/
stmt, err := db.Prepare("SELECT `user_name` FROM `users` WHERE `id` =  ?")
defer stmt.Close()
if err != nil {
	//错误处理
}
var userName string
//Scan() 将结果复制到userName
err = stmt.QueryRow(1).Scan(&userName)

fmt.Println(userName)

/*
   多行结果
*/
stmt, err := db.Prepare("SELECT `user_name` FROM `users` WHERE `age` =  ?")
defer stmt.Close()
if err != nil {
	//错误处理
}

rows, err := stmt.Query(年龄)
if err != nil {
	//错误处理
}

for rows.Next() {
	var userName string
	if err := rows.Scan(&userName); err != nil {
		//错误处理
	}
}
```
```go
/*
  插入操作
*/
stmt, err := db.Prepare("INSERT INTO `users` (`user_name`, `age`) VALUES(?, ?)")
defer stmt.Close()
if err != nil {
  //错误处理
}
stmt.Exec("名字", 年龄)
```
```go
/*
  事务
*/
tx, err := db.Begin()
if err != nil {
	//错误处理
}
defer tx.Rollback()

stmt, err := db.Prepare("")
defer stmt.Close()
if err != nil {
	//错误处理
}

stmt.Exec()
err = tx.Commit()
if err != nil {
	//错误处理
}
```


