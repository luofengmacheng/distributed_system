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

## 3 实现

基本想法是：

Master：启动Master后，当任务还没有完成时，获得一个没有执行任务的Worker，将任务分配给这个Worker，然后继续分配下一个任务。还要使用一个变量保存已经完成的任务数目，当完成了所有的任务，就可以进行接下来的工作。

Worker：等待任务，如果有任务到达，处理任务，然后返回结果。

下面简述下对代码的修改：

对代码做的修改：

1 在InitMapReduce中添加对Workers的创建

``` GO
mr.Workers = make(map[string]*WorkerInfo)
```

2 在WorkerInfo中添加了inUse bool表明这个工作者是否正在执行工作

3 修改RunMaster

``` GO
// 给worker分配了一个工作，工作的类型是job，工作号码是是JobNum，lock和comJob用于对完成的工作数目进行更新
func (mr *MapReduce) assignJob(worker *WorkerInfo, job JobType, JobNum int, lock *sync.Mutex, comJob *int) {
  var numOtherPhase int
  if job == Map {
    numOtherPhase = mr.nReduce
  } else if job == Reduce {
    numOtherPhase = mr.nMap
  }

  jobargs := &DoJobArgs{mr.file, job, JobNum, numOtherPhase}

  var jobreply DoJobReply
  ok := call(worker.address, "Worker.DoJob", jobargs, &jobreply) // 远程调用worker的Worker.DoJob
  if ok == false {
    fmt.Printf("call Worker.DoJob %d error", job)
  } else {
    worker.inUse = false;
    lock.Lock()
    *comJob++
    lock.Unlock()
  }
}

func (mr *MapReduce) RunMaster() *list.List {
  log.Printf("RunMaster start")
  // Your code here
  // 获得Worker的注册，从而确定Worker的存在
  // 并在Master保存它的相关信息
  workerAddr := <- mr.registerChannel
  _, ok := mr.Workers[workerAddr]
  if !ok {
    wkinfo := new(WorkerInfo)
    wkinfo.address = workerAddr
    wkinfo.inUse = false
    mr.Workers[workerAddr] = wkinfo
  }

  workerAddr = <- mr.registerChannel
  _, ok = mr.Workers[workerAddr]
  if !ok {
    wkinfo := new(WorkerInfo)
    wkinfo.address = workerAddr
    wkinfo.inUse = false
    mr.Workers[workerAddr] = wkinfo
  }

  log.Printf("go to cope with map jobs")

  var wkinfo *WorkerInfo

  comLock := new(sync.Mutex)
  workId := 0 // 记录当前分配的工作ID
  comJob := 0 // 记录已经完成的工作数目
  for {
    if workId >= mr.nMap { // 如果已经分发了所有的工作则推出循环
      log.Printf("send %d jobs to workers", workId)
      break
    }

    wkinfo = nil
    for _, wkinfo = range mr.Workers { // 获得一个没有执行工作的Worker
      if wkinfo.inUse == false {
        break
      }
    }

    if wkinfo != nil && wkinfo.inUse == false { // 如果找到了一个没有执行工作的Worker
      wkinfo.inUse = true
      go mr.assignJob(wkinfo, Map, workId, comLock, &comJob) // 就给它分配一个工作

      workId++ // 继续分配下一个工作
    }
  }

  for {
    if comJob >= mr.nMap { // 如果所有的工作都完成了，就推出循环，这个循环保证所有的Map任务完成
      break
    }

    // 这个循环用于等待所有的工作 完成，可是不知道为什么，在不加下一行这个休眠的函数时，最后两个工作总是得不到执行
    // assignJob函数获取了工作，但是没有执行。在下面加个time.Sleep()就行了，为什么呢？
    time.Sleep(time.Duration(1) * time.Second)
  }

  log.Printf("complete all map jobs")

  log.Printf("go to cope with reduce jobs")

  // 下面reduce的流程基本一样，就不赘述了
  workId = 0
  comJob = 0
  for {
    if workId >= mr.nReduce {
      log.Printf("send %d jobs to workers", workId)
      break
    }

    for _, wkinfo = range mr.Workers {
      if wkinfo.inUse == false {
        break
      }
    }

    if wkinfo != nil && wkinfo.inUse == false {
      wkinfo.inUse = true
      go mr.assignJob(wkinfo, Reduce, workId, comLock, &comJob)

      workId++
    }
  }

  for {
    if(comJob >= mr.nReduce) {
      break
    }

    time.Sleep(time.Duration(1) * time.Second)
  }

  log.Printf("complete all reduce jobs")

  return mr.KillWorkers()
}
```

### 4 测试

GO的单元测试文件名以test.go结尾，比如，这里mapreduce文件夹下面就有个test_test.go，这就是用于测试的文件。当在mapreduce目录下执行go test时，就会执行这个文件中函数名以Test开始的函数。因此，执行go test时，会依次执行TestBasic、TestOneFailure、TestManyFailures。对于实验1的第二部分，只要通过TestBasic就行。

对程序做以上修改，然后在mapreduce目录下执行go test > out，结果如图：

![](https://github.com/luofengmacheng/distributed_system/blob/master/pic/mapreduce2.png)

注意：这里没有直接输入go test，而是用了重定向，因为执行go test时，会在屏幕打印很多东西，而使用go test > out时，屏幕只会打印log.Printf()的内容，fmt.Printf()打印的内容会重定向到out文件中。因此，如果想只关注部分地方，可以用log.Printf()。比如，从这里可以看出，我将test_test.go中最后打印... Basic Passed的语句也变成log.Printf()。
