console
========
leaf 模型里面的控制台
> src/github.com/name5566/leaf/console/console.go
> src/github.com/name5566/leaf/console/command.go

## console.go
### 数据结构 变量
1. var server *network.TCPServer 
	定义了控制台server的类型，是个tcpserver
	
2. type Agent struct
	定义了控制台server 对应agent 的类型
```go
type Agent struct {
	conn   *network.TCPConn
	reader *bufio.Reader
}
```


### 函数
1.  func Init() 初始化
* 	加载参数
```go
func Init() {
	if conf.ConsolePort == 0 {
		return
	}
	//新生成一个tcp server 
	server = new(network.TCPServer)
	//设置本地端口
	server.Addr = "localhost:" + strconv.Itoa(conf.ConsolePort)
	//设置最大连接数
	server.MaxConnNum = int(math.MaxInt32)
	//每个连接同时最大写入数据量[]byte 个数
	server.PendingWriteNum = 100
	//初始化tpc server对应的newAgent 函数
	server.NewAgent = newAgent
	//服务启动
	server.Start()
}
```

2. func Destroy() 控制台销毁
```go
func Destroy() {
	if server != nil {
		//调用server close 函数 关闭tcpserver
		server.Close()
	}
}
```

3. func newAgent(conn *network.TCPConn) network.Agent 
	newAgent 函数
```go
//根据tcp连接 返回一个agent 
//network.Agent 是一个interface 
//console.Agent 满足该接口
func newAgent(conn *network.TCPConn) network.Agent {
	a := new(Agent)
	a.conn = conn
	a.reader = bufio.NewReader(conn)
	return a
}
```

3. func (a *Agent) Run() 
	满足network.Agent接口，定义run函数
```go
func (a *Agent) Run() {
	//循环
	for {
		
		//进入控制台之后， 向控制台agent 输出 console 前置 leaf# 
		if conf.ConsolePrompt != "" {
			a.conn.Write([]byte(conf.ConsolePrompt))
		}
		
		//从reader中获取数据 遇到\n 结束
		line, err := a.reader.ReadString('\n')
		if err != nil {
			break
		}
		
		//去除末尾换行符
		line = strings.TrimSuffix(line[:len(line)-1], "\r")

		//将接受到的数据按空格拆分
		args := strings.Fields(line)
		if len(args) == 0 {
			continue
		}
		//quit命令推出
		if args[0] == "quit" {
			break
		}
		
		//c 是一个Command
		//可以是CommandCPUProf  CommandHelp CommandProf ExternalCommand 4种可能
		var c Command
		//遍历 commands 如果收到的命令已经注册过 则退出
		for _, _c := range commands {
			if _c.name() == args[0] {
				c = _c
				break
			}
		}
		// 未找到命令
		if c == nil {
			a.conn.Write([]byte("command not found, try `help` for help\r\n"))
			continue
		}
		//找到命令 执行run 将输出送出
		output := c.run(args[1:])
		if output != "" {
			a.conn.Write([]byte(output + "\r\n"))
		}
	}
}
```
	
	
## 	command 注册
> src/github.com/name5566/leaf/module/skeleton.go
> func (s *Skeleton) RegisterCommand(name string, help string, f interface{})

通过scureCRT telnet 登陆之后 可以执行test命令


## 问题：
1. 为什么是telnet连接 ？ 
   从console.go 的init() 里面没有地方可以看出来是必须要telnet的

2. 建立一个client客户端，通过tcp连接到console端口 发送消息到后台没有效果
```go
func main() {

   //login()
	fmt.Println("start! .....")
	// 连接控制台
	consoleConn, err := net.Dial("tcp", "localhost:8012")
	if err != nil {
		panic(err)
	}
	fmt.Println("consoleConn! .....")

	var b []byte
	consoleConn.Read(b)
	fmt.Println("consoleConn.Read! .....",b)
	remsg := string(b[:])
	fmt.Println(remsg)

	cs :="test sadsd sddd"
	var conCommand []byte
	conCommand = []byte(cs)

	c := make([]byte, 2+len(conCommand))
	binary.BigEndian.PutUint16(c, uint16(len(conCommand)))
	copy(c[2:], conCommand)
	consoleConn.Write(c)
	fmt.Println("consoleConn.Write! .....",c)
	
}
```





	
