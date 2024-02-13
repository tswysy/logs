# Readme

# 本包以beego的logs模块为基础进行了自定义的修改。

* 将原来的Emergency等级更改为Fatal，Critical更改为Panic
* 调用Panic后将在写完日志后进行panic操作
* 调用Fatal后将在写完日志后退出程序
* 增加PanicLog函数，用于捕获未知的panic，写入日志并恢复。
* 增加Print方法，用来兼容输出GORM v1的日志
* 增加Printf方法，用来兼容输出GORM v2的日志

# 引擎配置设置

- console
        命令行输出，默认输出到`os.Stdout`：
    
        logs.SetLogger(logs.AdapterConsole, `{"level":1,"color":true}`)
        
    主要的参数如下说明：
    - level 输出的日志级别
    - color 是否开启打印日志彩色打印(需环境支持彩色输出)
    
- file

    设置的例子如下所示：

        logs.SetLogger(logs.AdapterFile, `{"filename":"test.log"}`)

    主要的参数如下说明：
    - filename 保存的文件名
    - maxlines 每个文件保存的最大行数，默认值 1000000
    - maxsize 每个文件保存的最大尺寸，默认值是 1 << 28, //256 MB
    - daily 是否按照每天 logrotate，默认是 true
    - maxdays 文件最多保存多少天，默认保存 7 天
    - rotate 是否开启 logrotate，默认是 true
    - level 日志保存的时候的级别，默认是 Trace 级别
    - perm 日志文件权限

- multifile

    设置的例子如下所示：

        logs.SetLogger(logs.AdapterMultiFile, `{"filename":"test.log","separate":["emergency", "alert", "critical", "error", "warning", "notice", "info", "debug"]}`)

    主要的参数如下说明(除 separate 外,均与file相同)：
    - filename 保存的文件名
    - maxlines 每个文件保存的最大行数，默认值 1000000
    - maxsize 每个文件保存的最大尺寸，默认值是 1 << 28, //256 MB
    - daily 是否按照每天 logrotate，默认是 true
    - maxdays 文件最多保存多少天，默认保存 7 天
    - rotate 是否开启 logrotate，默认是 true
    - level 日志保存的时候的级别，默认是 Trace 级别
    - perm 日志文件权限
    - separate 需要单独写入文件的日志级别,设置后命名类似 test.error.log


- conn

    网络输出，设置的例子如下所示：

        logs.SetLogger(logs.AdapterConn, `{"net":"tcp","addr":":7020"}`)

    主要的参数说明如下：
    - reconnectOnMsg 是否每次链接都重新打开链接，默认是 false
    - reconnect 是否自动重新链接地址，默认是 false
    - net 发开网络链接的方式，可以使用 tcp、unix、udp 等
    - addr 网络链接的地址
    - level  日志保存的时候的级别，默认是 Trace 级别

- smtp

    邮件发送，设置的例子如下所示：

        logs.SetLogger(logs.AdapterMail, `{"username":"beegotest@gmail.com","password":"xxxxxxxx","host":"smtp.gmail.com:587","sendTos":["xiemengjun@gmail.com"]}`)

    主要的参数说明如下：
    - username smtp 验证的用户名
    - password smtp 验证密码
    - host  发送的邮箱地址
    - sendTos   邮件需要发送的人，支持多个
    - subject   发送邮件的标题，默认是 `Diagnostic message from server`
    - level 日志发送的级别，默认是 Trace 级别

- ElasticSearch

    输出到 ElasticSearch:
```go
logs.SetLogger(logs.AdapterEs, `{"dsn":"http://localhost:9200/","level":1}`)
```

# 使用方法

## 通用方式
首先引入包：

```go
import (
    "github.com/tswysy/logs"
)
```

然后添加输出引擎（log 支持同时输出到多个引擎），这里我们以 console 为例，第一个参数是引擎名（包括：console、file、conn、smtp、es、multifile）

```go
logs.SetLogger("console")
```

添加输出引擎也支持第二个参数,用来表示配置信息，对于不同的引擎来说，其配置也是不同的。详细的配置请看上面介绍：

```go
logs.SetLogger(logs.AdapterFile,`{"filename":"project.log","level":7,"maxlines":0,"maxsize":0,"daily":true,"maxdays":10,"color":true}`)
```

然后我们就可以在我们的逻辑中开始任意的使用了：
```go
package main

import (
    "github.com/tswysy/logs"
)

func main() {
    logs.Debug("my book is bought in the year of ", 2016)
    logs.Info("this %s cat is %v years old", "yellow", 3)
    logs.Warn("json is a type of kv like", map[string]int{"key": 2016})
    logs.Error(1024, "is a very", "good game")
    logs.Panic("oh,crash")
}
```

## 多个实例
一般推荐使用通用方式进行日志，但依然支持单独声明来使用独立的日志
```go
package main

import (
    "github.com/tswysy/logs"
)

func main() {
    log := logs.NewLogger()
    log.SetLogger(logs.AdapterConsole)
    log.Debug("this is a debug message")
}
```

## 输出文件名和行号
日志默认输出调用的文件名和文件行号,如果你不期望输出调用的文件名和文件行号,可以如下设置
```go
logs.EnableFuncCallDepth(false)
```
关闭传入参数 false,默认是开启的.

如果你的应用自己封装了调用 logs 包,那么需要设置 SetLogFuncCallDepth(通用实例默认是3，单独实例默认是2),也就是直接调用的层级,如果你封装了多层,那么需要根据自己的需求进行调整.
```go
logs.SetLogFuncCallDepth(4)
```
## GORM使用
默认存在一个gormLogger实例，可以使用`GetGORMLogger()`获取，实例下包含两个logger对象
```go
type GORMLogger struct {
	consoleLogger *log.Logger // 终端打印对象
	beeLogger *BeeLogger // 文件保存对象
}

// 全局变量gormLogger
var gormLogger = GORMLogger{
	consoleLogger: log.New(os.Stdout, "\r\n", 0),
	beeLogger: beeLogger, // 默认将默认beeLogger实例存入，可以置空gormLogger.SetGormLogger(nil)，置空后将不再写入日志文件。sql日志等级为最低级Debug
}
```
