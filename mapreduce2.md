# 分发Map和Reduce任务

## 1 介绍

实验1任务1实现的是单机环境下统计单词个数，这里要实现分布式。主要的思想就是将RunSingle中两个for循环的DoMap和DoReduce交给Worker来做。

## 2 实验1任务2：分发Map和Reduce任务

实验1任务2主要是完成分布式环境下Master将Map任务和Reduce任务分发给Worker。

完成实验1任务2需要：
* 熟悉GO语言中的并发编程
* 熟悉GO语言中的网络编程，特别是RPC

首先，需要了解程序是如何分布式运行的，以及Master和Worker如何通信。

由前面可以知道main函数可以以三种方式运行，后两种就是将这个程序分别以Master身份和Worker身份运行。

``` GO
// Master (e.g., go run wc.go master x.txt localhost:7777)
// 以Master身份运行，且最后一个参数是本Master地址
// 因此os.Args[2] = "x.txt"，os.Args[3] = "localhost:7777"
mr := mapreduce.MakeMapReduce(5, 3, os.Args[2], os.Args[3])

// Worker (e.g., go run wc.go worker localhost:7777 localhost:7778 &)
// 以Worder身份运行，且倒数第二个参数是Master地址，最后一个参数是本Worker地址
// 因此os.Args[2] = "localhost:777"，os.Args[3] = "localhost:7778"
mapreduce.RunWorker(os.Args[2], os.Args[3], Map, Reduce, 100)
```

接下来，我们来分别分析Master和Worker的运行流程。

### 2.1 Master的运行流程

```
MakeMapReduce
	InitMapReduce 创建并初始化一个MapReduce结构体
	StartRegisterationServer
	go mr.Run()
```

下面主要来看看StartRegisterationServer和mr.Run()：

``` GO
func (mr *MapReduce) StartRegistrationServer() {
  rpcs := rpc.NewServer() // 新建服务器
  rpcs.Register(mr) // 注册服务对象，之后远程计算机就可以用过RPC调用mr的方法
  os.Remove(mr.MasterAddress)   // only needed for "unix"
  l, e := net.Listen("unix", mr.MasterAddress) // 设定地址和协议，并监听，这里用的是unix域套接字，可以很容易地转换成其它协议
  if e != nil {
    log.Fatal("RegstrationServer", mr.MasterAddress, " error: ", e)
  }
  mr.l = l

  // now that we are listening on the master address, can fork off
  // accepting connections to another thread.
  go func() {
    for mr.alive {
      conn, err := mr.l.Accept() // 接收远程计算机的连接
      if err == nil {
        go func() { // 用goroutine处理这个连接
          rpcs.ServeConn(conn)
          conn.Close()
        }()
      } else {
        DPrintf("RegistrationServer: accept error", err)
        break
      }
    }
    DPrintf("RegistrationServer: done\n")
  }()
}
```

``` GO
func (mr *MapReduce) Run() {
  fmt.Printf("Run mapreduce job %s %s\n", mr.MasterAddress, mr.file)

  mr.Split(mr.file) // 同单机版一样，对文件进行分割
  mr.stats = mr.RunMaster() // 运行Master
  mr.Merge() // 同单机版一样，对DoReduce的结果进行合并
  mr.CleanupRegistration() // 进行清理工作，实际是调用Shutdown，在Shutdown中将mr.alive设置为false

  fmt.Printf("%s: MapReduce done\n", mr.MasterAddress)

  // 向DoneChannel中写入true，表示Master工作完成
  // 之后在main函数中接收这个值，于是，Master结束
  mr.DoneChannel <- true
}
```

其中主要的函数就是mr.RunMaster，这也就是我们要完成的函数，对任务进行分发。

### 2.2 Worker的运行流程

``` GO
func RunWorker(MasterAddress string, me string,
               MapFunc func(string) *list.List,
               ReduceFunc func(string,*list.List) string, nRPC int) {
  DPrintf("RunWorker %s\n", me)
  wk := new(Worker) // 创建一个Worker结构体
  wk.name = me // 初始化结构体中的成员
  wk.Map = MapFunc
  wk.Reduce = ReduceFunc
  wk.nRPC = nRPC
  rpcs := rpc.NewServer() // 下面的行为跟Master类似
  rpcs.Register(wk)
  os.Remove(me)   // only needed for "unix"
  l, e := net.Listen("unix", me)
  if e != nil {
    log.Fatal("RunWorker: worker ", me, " error: ", e)
  }
  wk.l = l
  Register(MasterAddress, me) // 向Master告知自己的存在

  // DON'T MODIFY CODE BELOW
  for wk.nRPC != 0 {
    conn, err := wk.l.Accept()
    if err == nil {
      wk.nRPC -= 1
      go rpcs.ServeConn(conn)
      wk.nJobs += 1
    } else {
      break
    }
  }
  wk.l.Close()
  DPrintf("RunWorker %s exit\n", me)
}
```

其中主要的函数就是Register，这个函数告知Master自己的存在，这个函数通过RPC调用了Master的Register。

### 3 实现

Master：启动Master后，当任务还没有完成时，如果有Worker还没有分配任务，那么就将这个任务交给该Worker。
Worker：等待任务，如果有任务到达，处理任务，然后返回结果。
