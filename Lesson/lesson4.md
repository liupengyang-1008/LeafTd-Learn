# **lesson4**  2018-06-24
# Leaftd  H5 注册消息调试

1. 消息初始化【注册】
	src/github.com/name5566/leaf/network/json/json.go
	
	type Processor struct {
	msgInfo map[string]*MsgInfo			//
	}
	
	type MsgInfo struct {
	msgType       reflect.Type			//消息结构体类型名称 类似 ChooseRole: *msg.ChooseRole
	msgRouter     *chanrpc.Server		//在注册路由时绑定需要转发的rpcserver
	msgHandler    MsgHandler			
	msgRawHandler MsgHandler		
	}
	
	func NewProcessor() *Processor {
	p := new(Processor)
	p.msgInfo = make(map[string]*MsgInfo)
	return p
	}
	
	func (p *Processor) Register(msg interface{}) {}
	msg 必须是指针类型
	检查msg 已经在MsgInfo中是否已经存在，如果不存在则存入MsgInfo的map中
	最终在Processor注册后的内容
	｛
	  map[ChooseRole] {
	msgType       :*msg.ChooseRole
	msgRouter     : game.ChanRPC
	msgHandler    : nil			
	msgRawHandler : nil		
	}
	｝
	
	
	msg.Processor 的初始化在Module的OnInit函数中
	src/LpySever/gate/internal/module.go
	
	OnInit函数 在每个Module的注册时被运行，即leaf.Run {gate.Module }  
	
2. 注册路由
	在gate module 初始化时，会注册路由
	src/LpySever/gate/router.go	：func init() 
	通过SetRouter 函数将消息类型与chanrpc server 绑定起来，
	chanrpc server是分模块的，绑定之后，当消息来临时 通过绑定的发送给对应的rpcserver
	
	此处路由注册与上面的消息注册类似，都注册到Processor的结构体当中，
	只不过消息注册时对MsgInfo 中的msgType赋值，路由注册时，对msgRouter 赋值，注册路由时必须消息先注册。
	
3. 消息的分发
	在gate module启动时，	cluster.Init()部分会开启TCP 监听
	通过server.ln.Accept() 接受消息
	Accept() 是标准库net/http/server.go 中的内容，属于 tcpKeepAliveListener 结构
	
	接受之后建立tcp 连接 生成agent  并执行agent.Run()
	这里的agent是 gate里面的
	src/github.com/name5566/leaf/gate/gate.go
	
	agent.Run() 获得接受到的数据 Unmarshal 解码
	然后a.gate.Processor.Route(msg, a) 发送路由
	由于是json 格式数据，Route 实现在
	src/github.com/name5566/leaf/network/json/json.go
	
	通过反射获得消息类型，到processer 中获得msgRouter
	执行i.msgRouter.Go(msgType, msg, userData)
	这里的Go是 chanrpc 中的go函数
	即通过消息类型找到chanrpc 中的绑定函数，并执行。
	参数 msgType 是消息类型 ； msg 是消息内容； userData 由gate传入，是agent 即tcp连接的客户端
	
	chanrpc.Go 执行时将msgType 作为f ，msg 和userData 作为 args 当作CallInfo 传入ChanCall
	
	在module.run 中，时刻监听channel ChanCall，如果有消息则执行 chanrpc.Exec 
	
	

	
3.  消息处理
	通过上面的内容。 对于ChooseRole 绑定的是game module 的chanrpc。 所以会被转发到game
	
	game module 在初始化时，会将自己module 需要处理的消息函数 注册到chanrpc server中
	src/LpySever/game/internal/handler.go
	消息注册 消息绑定
	handler(&Protocol.ChooseRole{}, handleChooseRole)
	handler通过调用skeleton.RegisterChanRPC 向chanrpc server 注册
	&Protocol.ChooseRole{} 是消息类型，handleChooseRole 是消息处理函数 包含消息处理的具体逻辑
	
	在chanprc 注册之后，chanrpc.Exec  就可以正常执行
	
	

	
	
# 重要内容
1. 熟悉反射
	https://mp.weixin.qq.com/s/9CgwzZlXhmZn2SO4XDljoQ 	
	
2. 每个模块都要有个data	manager
	
	
# 课后习题
1. 	leaf框架 SetHandler 函数使用；应用到逻辑层哪里？
	SetHandler 是将消息和一个函数【msgHandler】绑定，绑定函数没有返回值 
	同时在MsgInfo 中与msgRouter 一起绑定
	在Route 函数中，远程执行rpc之前，先执行msgHandler ，
	可以对上送的msg数据预处理，包含校验\加工 等等
	
	
2. 消息流程的流程图;标注函数最好，主要是让大家熟悉消息的机制
	见附件
3. 项目中的 a := args[1].(gate.Agent) ； (gate.Agent) 什么含义  ；a:=ProtocolData["Itype"].(float64)   a为何是int型的？？
	在Route中可以看出，在执行rpc调用的时候，args有2个参数 是【msg, userData】，即args[1]是userData，
	而是userData，在gate: func (a *agent) Run()中，可以看出userData 是agent struct
	故a := args[1].(gate.Agent) 其实是将args[1] 这个interface 强制转换为 agent struct 类型，便于后续解析，处理。
	
	

  熟悉interface的接口如何取值等

熟悉 反射机制  接口概念
	
# 视频下载链接
链接: https://pan.baidu.com/s/1ABfByFdrosZyqp9nBa2lfA 密码: tj93