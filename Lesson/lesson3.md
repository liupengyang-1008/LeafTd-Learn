# 项目结构

***
消息流程的定义

所有的消息定义在下面
src/LollipopGo/msg/protocolfile/protocol.go
消息注册
src/LollipopGo/msg/msg.go


***
1. 服务启动
src/LollipopGo/main.go
加载配置之后 leaf.Run 以login.Module 为例

2. leaf.Run
src/github.com/name5566/leaf/leaf.go 	
> func Run(mods ...module.Module)
	2.1 注册module  
	src/github.com/name5566/leaf/module/module.go	
> 	module.Register(mods[i])
	主要是new(module) 分配内存空间
	
	2.2 module.Init() 
	执行每个module的	OnInit()  函数
	对于login.Module 在 src/LollipopGo/login/internal/module.go 
> 	func (m *Module) OnInit()
	初始化一个skeleton 给login.Module
	
	2.3 cluster.Init() 
	主要完成网络相关初始化
> 	src/github.com/name5566/leaf/cluster/cluster.go	
	2.3.1 加载 conf 参数 到tcpserver
	参数都定义在 src/github.com/name5566/leaf/conf/conf.go
	在启动时都已加载 到conf当中
	
	2.3.2 server.Start() 
> 	src/github.com/name5566/leaf/cluster/cluster.go
	对应func (server *TCPServer) Start()
> 	src/github.com/name5566/leaf/network/tcp_server.go
	初始话server之后go server.run()
	
	在 func (server *TCPServer) run() 当中循环监控网络连接
	当有新的连接进来时 即生成一个agent :
> 	type Agent struct :src/github.com/name5566/leaf/cluster/cluster.go
	然后go func() :agent.Run() 
	agent.Run()  是个空函数 【func (a *Agent) Run() {}】 不清楚到底哦怎么run的
>   src/github.com/name5566/leaf/cluster/cluster.go	
	
	
	2.3.4 client.Start()
	与server类似，只不过client 会有很多个，是个遍历 
	初始化之后调用func (client *TCPClient) Start() 
> 	src/github.com/name5566/leaf/network/tcp_client.go	
 	后面的没细看
	
	2.4 console.Init()
	使用控制台端口在本地端起一个tcpserver
	server.Start() 与2.3.2类似
	
	2.5 接受关闭信号，关闭服务器
	这里没看懂,不知道怎么和ctrl + c关联的
	
	
3. H5_Client
> src/H5_Client/index.html
	在function login() 中，注册UserLogin信息，转化为json后发送到tcp端口
	doSend(goServerJson);
		websocket1.send(message);
		
4. server 端login 接受
   	4.1 gate moudle 负责监听所有消息，并注册有路由负责转发消息
> 	src/LollipopGo/gate/router.go
	func init() 中有注册向UserLogin 转发的路由
	`	msg.Processor.SetRouter(&Protocol.UserLogin{}, login.ChanRPC)`	
	通过SetRouter函数 将消息类型和消息通道绑定
	
	4.2 消息处理
	4.2.1 向当前模块（login 模块）注册 Protocol.UserLogin 消息的消息处理函数 handleTest
	func init()
> 	src/LollipopGo/login/internal/handler.go
	
	4.2.2 消息处理函数 handleTest
	func handleTest(args []interface{}) 
> 	src/LollipopGo/login/internal/handler.go
	 将收到的数据转化为Protocol.UserLogin类型
	上送的数据和注册的类型都是	UserLogin所以可以成功处理
	可直接访问UserLogin结构体下的内容
	
	4.2.3 消息返回
	可以根据业务功能组成返回消息，调用WriteMsg()函数返回
	WriteMsg()是gate tcp连接agent的方法 
	 	
	


***
以上是总结，会有遗漏，欢迎大家补充	
	




# 作业
* 完成一个选择角色的消息
在 src/LollipopGo/game/internal/handler.go注册

* 配置mysql数据库
实现和外网数据库的连接

* 脚本自动实现赋予执行权限

# 其他
html5 开发工具 HBuilder
http://www.dcloud.io/

