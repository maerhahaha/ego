[TOC]
## 说明
通用的GOLANG公共库,拒绝困扰程序员，提高生产力。

## 特性
- 提供简单的HTTP服务启动方式
- 提供简单的日志管理器
- 自由的集成组件,拒绝臃肿(比如PHP某框架,不点名了)
- 中间件(记录请求日志、JWT认证)
- 集成Apollo,支持热更新
- 集成Consul,支持服务注册、发现、注销
- Mysql,Redis连接池
- 集成prometheus监控

## 安装

```
go get -u github.com/ebar-go/ego
```

## 模块
### http
基于gin框架的http服务器模块
- 启动web服务

```go
package main
import (
	"github.com/ebar-go/ego/http"
	"github.com/gin-gonic/gin"
	"fmt"
    "github.com/ebar-go/ego/helper"
	)

func main() {
    server := http.NewServer()
    // 添加路由
    server.Router.GET("/test", func(context *gin.Context) {
        fmt.Println("hello,world")
    })
    
    helper.CheckErr("StartServer", server.Start(), true)
}
```

#### 中间件
集成JWT,跨域等中间件
- JWT
```go
package main
import (
	"github.com/ebar-go/ego/http"
	"github.com/gin-gonic/gin"
	"fmt"
	"github.com/ebar-go/ego/http/middleware"
	"github.com/ebar-go/ego/helper"
	)
func main() {
    server := http.NewServer()
    server.SetJwtKey([]byte("jwt_key"))
    // 添加路由
    server.Router.GET("/test", func(context *gin.Context) {
        fmt.Println("hello,world")
    })
    
    api := server.Router.Group("/api")
    api.Use(middleware.JWT)
    {
    	api.GET("/user", func(context *gin.Context) {
    		
    	    fmt.Println("获取用户信息")
    	    fmt.Println(middleware.GetCurrentClaims(context))
    	})
    }
    helper.CheckErr("StartServer", server.Start(), true)
}
```

### cache
推荐使用 go-cache

### config
配置项,集成Apollo配置
- Apollo

```go
package main
import (
	"github.com/ebar-go/ego/component/apollo"
    "os"
	"fmt"
	"github.com/ebar-go/ego/helper"
)
func main() {
    conf := apollo.Conf{
    	AppId: "open-api",
    	Cluster: "local",
    	Ip: "192.168.0.19:8080",
    	Namespace: "application",
    }
    helper.CheckErr("InitApollo", apollo.Init(conf), true)
    
    // 获取配置
    logFilePath := apollo.GetStringValue("LOG_FILE","/var/tmp")
    fmt.Println(logFilePath)
    // 另外，可以使用定时任务，监听配置变更
}

```
更多方法请查看测试用例

### consul
微服务(SOA),集成consul组件
- 服务注册

```go
package main
import (
	"github.com/ebar-go/ego/component/consul"
	"fmt"
	"github.com/ebar-go/ego/helper"
)
func main() {
    config := consul.DefaultConfig()
    // 指定consul地址
    config.Address = "192.168.0.222:8500"
    client := &consul.Client{
        Config:config,
    }
    // 获取本机IP
    ip, err := helper.GetLocalIp()  
    if err != nil {
    	panic(err)
    }
    registration := consul.NewServiceRegistration()
    registration.ID = "go-demo-2"
    registration.Name = "project-demo"
    registration.Port = 8088
    registration.Tags = []string{"project-demo"}
    registration.Address = ip
    check := consul.NewServiceCheck()
    // 指定服务检查url
    check.HTTP = fmt.Sprintf("http://%s:%d%s", registration.Address, registration.Port, "/check")
    check.Timeout = "3s"
    check.Interval = "3s"
    check.DeregisterCriticalServiceAfter = "30s" //check失败后30秒删除本服务
    registration.Check = check
    err = client.Register(registration)
}
```
- 服务发现

```go
package main
import (
	"github.com/ebar-go/ego/component/consul"
	"fmt"
	"github.com/ebar-go/ego/helper"
)
func main() {
    config := consul.DefaultConfig()
    // 指定consul地址
    config.Address = "192.168.0.222:8500"
    client := &consul.Client{
        Config:config,
    }
    items, err := client.Discover("project-demo")
    if err != nil {
    	panic(err)
    }
    service, err := client.LoadBalance(items)
    fmt.Println(service.GetHost())
}
```

更多方法请查看测试用例

### helper
公共库

```go
package main
import (
    "github.com/ebar-go/ego/helper"
    "fmt"
)
func main() {//
    // 获取当前时间
    fmt.Println("获取当前时间:" , helper.GetTimeStr())
    helper.Debug("打印调试")
}
```

### log
日志管理器
- 系统日志

```go
package main
import (
       	"github.com/ebar-go/ego/log"
       	"os"
       )
func main() {
    log.App().Info("test", log.Context{"a":1})
    log.App().Debug("test", log.Context{"a":1})
    log.App().Warn("test", log.Context{"a":1})
}
```
- 自定义

```go
package main
import (
       	"github.com/ebar-go/ego/log"
       	"os"
       )
func main() {
    logger := log.New()
    logger.Debug("test debug", log.Context{"name":"123"})
}
```


### HTTP请求客户端
提供官方的http包、fasthttp(推荐使用),kong网关的http客户端
```go
package main
import (
       	"github.com/ebar-go/ego/component/mysql"
       	"github.com/ebar-go/ego/http/client"
       	"github.com/ebar-go/ego/http/client/request"
       	"os"
       	"fmt"
       )
func main() {
	// 官方http,支持长连接
	httpClient := client.NewHttpClient()
	
	// fasthttp
	// httpClient := client.NewFastHttpClient()
	
	// kong
	// httpClient := client.NewKongClient()
	// 设置其他参数...
	req := httpClient.NewRequest(request.Param{
		Url: "http://localhost:8080/test",
		Method: request.Get,
	})
	
	resp, err := httpClient.Execute(req)
	fmt.Println(resp, err)
}
```

### mysql
集成的Gorm,使用连接池

```go
package main
import (
       	"github.com/ebar-go/ego/component/mysql"
       	"github.com/ebar-go/ego/helper"
       	"os"
       )
func main() {
    conf := mysql.Conf{
    	Name: "test",
    	Host: "127.0.0.1",
    	Port: 3306,
    	Default: true,
    	LogMode:true,
    }
    
    helper.CheckErr("InitMysqlPool", mysql.InitPool(conf), true)
    
    conn := mysql.GetConnection()
    conn.DB().Ping()
    defer func() {
    	mysql.CloseConnectionGroup()
    }()
    
}
```

### redis
集成的go-redis,使用连接池

```go
package main
import (
       	"github.com/ebar-go/ego/component/redis"
       	"github.com/ebar-go/ego/helper"
       	"os"
       	"fmt"
       )
func main() {
    conf := redis.Conf{
    	Host: "192.168.0.222",
    	Port: 6379,
    }
    
    helper.CheckErr("InitRedisPool", redis.InitPool(conf), true)
    
    conn := redis.GetConnection()
    if err := conn.Set("key", "value", 0).Err(); err != nil {
    	fmt.Println(err)
    }
    
    val, err := conn.Get("key").Result()
    fmt.Println("key", val, err)
}
```

### 对接prometheus监控
监控Mysql

```go
package main
import (
       	"github.com/ebar-go/ego/component/mysql"
       	"github.com/ebar-go/ego/component/prometheus"
       	"github.com/ebar-go/ego/helper"
       	"github.com/ebar-go/ego/http"
       	"os"
       )
func main() {
    conf := mysql.Conf{
    	Dsn: "root:123456@tcp(127.0.0.1:3306)/test?charset=utf8&parseTime=True&loc=Local",
    	LogMode:true,
    }
    
    helper.CheckErr("InitMysqlPool", mysql.InitPool(conf), true)
    
    conn := mysql.GetConnection()
    prometheus.ListenMysql(conn, "server")
    
    server := http.NewServer()
    
    server.Router.GET("/metrics", prometheus.Handler)
    
    helper.CheckErr("StartServer", server.Start(), true)
    
}
```

### test 单元测试


## TODO
- 参数验证器