# go-websocket
go websocket demo
## 代码流程
- 搭建一个http服务，handler为wsHandler
- 初始化websocket.Upgader
- 升级http连接为websocket连接
- 使用websocket连接进行通信
## 线程安全封装
原始websocket的ReadMessage和WriteMessage方法是线程不安全的，如果在一个handler中需要启动多个goroutine进行消息推送或读取，那么可能产生重复读取或推送的情况。
### impl/connection.go
要解决线程不安全的问题，最直接的方式就是不要直接用多个goroutine去访问原生API，而是读写都只用一个goroutine然后将读到的数据送入channel，其他的goroutine都从channel中进行读写，由于channel是线程安全的，则封装过后的websocket连接也是线程安全的。
> go conn.readLoop()
go conn.writeLoop()
就负责进行统一的读写

channel读写存在一个阻塞的问题，如果由于websocket连接关闭导致channel中没有数据则调用者将始终阻塞在channel读写中，所以需要在连接关闭时同时关闭channel并且在封装后的接口中使用select语句进行监听和退出。
> func (conn *Connection) ReadMessage() (data []byte, err error)
func (conn *Connection) WriteMessage(data []byte) ( err error)
使用了select语句进行channel关闭的监听和退出

channel关闭只能执行一次，但readLoop和writeLoop都有可能因为网络问题关闭连接和channel，所以要对channel的close做标记和判断，同时需要加锁来保证线程安全。
