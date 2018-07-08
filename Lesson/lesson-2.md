1. 交叉编译
	win平台需要将linux版本覆盖win代码
	
2. channel rpc
   	同步
	异步
	go 模式


	函数有3种注册方式
	通过exec开启注册函数【执行】
	
3.  下节开始通过断点调试模式
	下节协议网络模型 
	预习H5 ：src/H5_Client/index.html
	websocket  -- 通讯
	调试其他模块
	联调net模块 src/github.com/name5566/leaf/network


### 作业：
1.  自己建立一个rpc通信的demo
   外网部署一个rpc sever
   rpc 通信a-b-c
   通过守护进程运行起来 shell脚本

2. 预习H5 ：src/H5_Client/index.html
	websocket  -- 通讯


### 附注：
外网每人一个端口

周三【13号】之前自己摸索
周四晚上彬哥在  23点 部署到自己的目录下 用自己的端口




# RPC:
* RPC server
RPC server 是一个结构体
	type Server struct {
	// id -> function
	//
	// function:
	// func(args []interface{})
	// func(args []interface{}) interface{}
	// func(args []interface{}) []interface{}
	// 函数的map 
	// 每个函数的参数可以是任何值 ，返回值可以是任何值
	functions map[interface{}]interface{}
	
	// ChanCall 结构的channel 
	ChanCall  chan *CallInfo
	}
	
* CallInfo
CallInfo 调用信息

`type CallInfo struct {`
`	// function 表示调用后后被什么执行什么函数`
`	// f 的值只能是server.functions`
`	f       interface{}`
`	// 执行f 时传入的参数`
`	args    []interface{}`
`	// RetInfo 的channel`
`	chanRet chan *RetInfo`

`	cb      interface{}`
`}`
	

*	RetInfo
returen info 返回信息
`type RetInfo struct {`
`	// nil`
`	// interface{}`
`	// []interface{}`
`	ret interface{}`

`	err error`

	回调函数
`	// callback:`
`	// func(err error)`
`	// func(ret interface{}, err error)`
`	// func(ret []interface{}, err error)`
`	cb interface{}`
`}`

* Client
客户端结构
`type Client struct {`
	客户端指向的服务器
`	s               *Server`
	同步返回
`	chanSyncRet     chan *RetInfo`
	异步返回
`	ChanAsynRet     chan *RetInfo`
	
`	pendingAsynCall int`
`}`

* NewServer

`func NewServer(l int) *Server {`
`	s := new(Server)`
`	s.functions = make(map[interface{}]interface{})`
`	s.ChanCall = make(chan *CallInfo, l)`
`	return s`
`}`

* assert
	将一个对象强制转换为切片
`func assert(i interface{}) []interface{} {`
`	if i == nil {`
`		return nil`
`	} else {`
`		return i.([]interface{})`
`	}`
`}`


* Register 方法
  注册一个rpcserver
参数 函数id 和函数本身
`// you must call the function before calling Open and Go`
`func (s *Server) Register(id interface{}, f interface{}) {`
	id 标识
	f  rpc server 功能处理函数 
`	switch f.(type) {`
`	case func([]interface{}):`
`	case func([]interface{}) interface{}:`
`	case func([]interface{}) []interface{}:`
`	default:`
`		panic(fmt.Sprintf("function id %v: definition of function is invalid", id))`
`	}`
    根据id在server里的function map中查找，如果找到说明注册过了
`	if _, ok := s.functions[id]; ok {`
`		panic(fmt.Sprintf("function id %v: already registered", id))`
`	}`
	将传入的id 和函数放入function map中
`	s.functions[id] = f`
`}`


* ret 方法
  参数 CallInfo - 调用信息 RetInfo -返回信息
  	   
`func (s *Server) ret(ci *CallInfo, ri *RetInfo) (err error) {`
	nil channel ？
`	if ci.chanRet == nil {`
`		return`
`	}`

`	defer func() {`
`		if r := recover(); r != nil {`
`			err = r.(error)`
`		}`
`	}()`

`	ri.cb = ci.cb`
`	ci.chanRet <- ri`
`	return`
`}`

* exec 执行

`func (s *Server) exec(ci *CallInfo) (err error) {`
`	defer func() {`
`		if r := recover(); r != nil {`
`			if conf.LenStackBuf > 0 {`
`				buf := make([]byte, conf.LenStackBuf)`
`				l := runtime.Stack(buf, false)`
`				err = fmt.Errorf("%v: %s", r, buf[:l])`
`			} else {`
`				err = fmt.Errorf("%v", r)`
`			}`
			执行ret方法
`			s.ret(ci, &RetInfo{err: fmt.Errorf("%v", r)})`
`		}`
`	}()`

`	// execute`
`	switch ci.f.(type) {`
`	case func([]interface{}):`
`		ci.f.(func([]interface{}))(ci.args)`
`		return s.ret(ci, &RetInfo{})`
`	case func([]interface{}) interface{}:`
`		ret := ci.f.(func([]interface{}) interface{})(ci.args)`
`		return s.ret(ci, &RetInfo{ret: ret})`
`	case func([]interface{}) []interface{}:`
`		ret := ci.f.(func([]interface{}) []interface{})(ci.args)`
`		return s.ret(ci, &RetInfo{ret: ret})`
`	}`

`	panic("bug")`
`}`

* Exec 
执行exec 公有
`func (s *Server) Exec(ci *CallInfo) {`
`	err := s.exec(ci)`
`	if err != nil {`
`		log.Error("%v", err)`
`	}`
`}`


`// goroutine safe`
向调用信息队列中传值 ，并注册
`func (s *Server) Go(id interface{}, args ...interface{}) {`
`	f := s.functions[id]`
`	if f == nil {`
`		return`
`	}`

`	defer func() {`
`		recover()`
`	}()`

	将参数传入server 的 ChanCall  channel
`	s.ChanCall <- &CallInfo{`
`		f:    f,`
`		args: args,`
`	}`
`}`

`// goroutine safe`
`func (s *Server) Call0(id interface{}, args ...interface{}) error {`
`	return s.Open(0).Call0(id, args...)`
`}`

`// goroutine safe`
`func (s *Server) Call1(id interface{}, args ...interface{}) (interface{}, error) {`
`	return s.Open(0).Call1(id, args...)`
`}`

`// goroutine safe`
`func (s *Server) CallN(id interface{}, args ...interface{}) ([]interface{}, error) {`
`	return s.Open(0).CallN(id, args...)`
`}`

关闭调用队列
`func (s *Server) Close() {`
`	close(s.ChanCall)`

`	for ci := range s.ChanCall {`
`		s.ret(ci, &RetInfo{`
`			err: errors.New("chanrpc server closed"),`
`		})`
`	}`
`}`

`// goroutine safe`
返回一个客户端 ，并将客户端和server关联
`func (s *Server) Open(l int) *Client {`
`	c := NewClient(l)`
`	c.Attach(s)`
`	return c`
`}`

新建客户端 参数返回channel 的缓存值
`func NewClient(l int) *Client {`
`	c := new(Client)`
`	c.chanSyncRet = make(chan *RetInfo, 1)`
`	c.ChanAsynRet = make(chan *RetInfo, l)`
`	return c`
`}`

将客户端和server关联
`func (c *Client) Attach(s *Server) {`
`	c.s = s`
`}`


* call Client
客户端向服务器发起通信请求
block为false 时 如果消息队列阻塞会报错
`func (c *Client) call(ci *CallInfo, block bool) (err error) {`
`	defer func() {`
`		if r := recover(); r != nil {`
`			err = r.(error)`
`		}`
`	}()`

`	if block {`
`		c.s.ChanCall <- ci`
`	} else {`
`		select {`
`		case c.s.ChanCall <- ci:`
`		default:`
`			err = errors.New("chanrpc channel full")`
`		}`
`	}`
`	return`
`}`

* f Client
判定客户端联通的服务器是否可以实现 f 功能
如果存在 则获取f 功能函数
`func (c *Client) f(id interface{}, n int) (f interface{}, err error) {`
`	if c.s == nil {`
`		err = errors.New("server not attached")`
`		return`
`	}`

`	f = c.s.functions[id]`
`	if f == nil {`
`		err = fmt.Errorf("function id %v: function not registered", id)`
`		return`
`	}`

`	var ok bool`
`	switch n {`
`	case 0:`
`		_, ok = f.(func([]interface{}))`
`	case 1:`
`		_, ok = f.(func([]interface{}) interface{})`
`	case 2:`
`		_, ok = f.(func([]interface{}) []interface{})`
`	default:`
`		panic("bug")`
`	}`

`	if !ok {`
`		err = fmt.Errorf("function id %v: return type mismatch", id)`
`	}`
`	return`
`}`



`func (c *Client) Call0(id interface{}, args ...interface{}) error {`
`	f, err := c.f(id, 0)`
`	if err != nil {`
`		return err`
`	}`

`	err = c.call(&CallInfo{`
`		f:       f,`
`		args:    args,`
`		chanRet: c.chanSyncRet,`
`	}, true)`
`	if err != nil {`
`		return err`
`	}`

`	ri := <-c.chanSyncRet`
`	return ri.err`
`}`

`func (c *Client) Call1(id interface{}, args ...interface{}) (interface{}, error) {`
`	f, err := c.f(id, 1)`
`	if err != nil {`
`		return nil, err`
`	}`

`	err = c.call(&CallInfo{`
`		f:       f,`
`		args:    args,`
`		chanRet: c.chanSyncRet,`
`	}, true)`
`	if err != nil {`
`		return nil, err`
`	}`

`	ri := <-c.chanSyncRet`
`	return ri.ret, ri.err`
`}`

`func (c *Client) CallN(id interface{}, args ...interface{}) ([]interface{}, error) {`
`	f, err := c.f(id, 2)`
`	if err != nil {`
`		return nil, err`
`	}`

`	err = c.call(&CallInfo{`
`		f:       f,`
`		args:    args,`
`		chanRet: c.chanSyncRet,`
`	}, true)`
`	if err != nil {`
`		return nil, err`
`	}`

`	ri := <-c.chanSyncRet`
`	return assert(ri.ret), ri.err`
`}`

`func (c *Client) asynCall(id interface{}, args []interface{}, cb interface{}, n int) {`
`	f, err := c.f(id, n)`
`	if err != nil {`
`		c.ChanAsynRet <- &RetInfo{err: err, cb: cb}`
`		return`
`	}`

`	err = c.call(&CallInfo{`
`		f:       f,`
`		args:    args,`
`		chanRet: c.ChanAsynRet,`
`		cb:      cb,`
`	}, false)`
`	if err != nil {`
`		c.ChanAsynRet <- &RetInfo{err: err, cb: cb}`
`		return`
`	}`
`}`

`func (c *Client) AsynCall(id interface{}, _args ...interface{}) {`
`	if len(_args) < 1 {`
`		panic("callback function not found")`
`	}`

`	args := _args[:len(_args)-1]`
`	cb := _args[len(_args)-1]`

`	var n int`
`	switch cb.(type) {`
`	case func(error):`
`		n = 0`
`	case func(interface{}, error):`
`		n = 1`
`	case func([]interface{}, error):`
`		n = 2`
`	default:`
`		panic("definition of callback function is invalid")`
`	}`

`	// too many calls`
`	if c.pendingAsynCall >= cap(c.ChanAsynRet) {`
`		execCb(&RetInfo{err: errors.New("too many calls"), cb: cb})`
`		return`
`	}`

`	c.asynCall(id, args, cb, n)`
`	c.pendingAsynCall++`
`}`

`func execCb(ri *RetInfo) {`
`	defer func() {`
`		if r := recover(); r != nil {`
`			if conf.LenStackBuf > 0 {`
`				buf := make([]byte, conf.LenStackBuf)`
`				l := runtime.Stack(buf, false)`
`				log.Error("%v: %s", r, buf[:l])`
`			} else {`
`				log.Error("%v", r)`
`			}`
`		}`
`	}()`

`	// execute`
`	switch ri.cb.(type) {`
`	case func(error):`
`		ri.cb.(func(error))(ri.err)`
`	case func(interface{}, error):`
`		ri.cb.(func(interface{}, error))(ri.ret, ri.err)`
`	case func([]interface{}, error):`
`		ri.cb.(func([]interface{}, error))(assert(ri.ret), ri.err)`
`	default:`
`		panic("bug")`
`	}`
`	return`
`}`

`func (c *Client) Cb(ri *RetInfo) {`
`	c.pendingAsynCall--`
`	execCb(ri)`
`}`

`func (c *Client) Close() {`
`	for c.pendingAsynCall > 0 {`
`		c.Cb(<-c.ChanAsynRet)`
`	}`
`}`

`func (c *Client) Idle() bool {`
`	return c.pendingAsynCall == 0`
`}`

`	`	`




