# 实验2任务1：Viewservice

## 1 实验介绍

在这个实验中，要用主备副本机制实现容错的K/V服务。为了保证所有的部分同意哪个服务器是主服务器，哪个服务器是备份服务器，我们引入了一种管理服务器，称之为viewservice。viewservice监控服务器是否可以使用。如果当前的主服务器或者备份服务器发生故障，viewservice会选择一个服务器代替它。客户端向viewservice查询当前的主服务器。服务器与viewservice协同工作，保证在同一时刻至多有一个主服务器工作。

如果主服务器宕机了，viewservice会将备份服务器变成主服务器。如果备份服务器宕机了，或者它变成了主服务器，并且现在还有可用的服务器，viewservice就会将它变成备份服务器。主服务器会将它的完整的数据库发送给新的备份服务器，然后将接下来的Put操作发送给备份服务器，以保证备份服务器的K/V数据库跟主服务器一样。

也就是说，主服务器必须将Get和Put操作发送到备份服务器，在响应客户端之前必须等待备份服务器的响应。这就防止了两个服务器都以主服务器运行。比如：S1是主服务器，S2是备份服务器。viewservice错误地认为S1死掉了，然后将S2变成主服务器。如果某个客户端仍然认为S1是主服务器，并将操作发送给S1，S1就会将操作发送给S2，S2会返回一个错误来告诉S1，它不是备份服务器。S1就会向客户端发送一个错误，告知客户端S1可能不是主服务器；然后，客户端就会向viewservice询问正确的主服务器，然后向主服务器请求操作。

发生故障的K/V服务器可能重启，但是它没有副本数据的拷贝。也就是说，K/V服务器会将数据保存在内存中，而不是磁盘上。将数据保存在内存中的结果是：如果没有备份服务器，主服务器宕机了，然后重启，它就不能作为主服务器(可能是因为没有之前的数据吧)。

## 2 Viewservice介绍

viewservice包含一系列编号的views，每个都是跟主备服务器相关。一个view包含一个整数和主备服务器的身份(host:port)。

在一次view的主服务器必须是前一个view的主服务器或者备份服务器。这能够保证，保存K/V服务器的状态。一个例外是：当viewservice第一次启动时，它可以接受任何服务器作为主服务器。在一次view的备份服务器可以是任何服务器，或者当没有服务器可用时，就没有备份服务器(以空串表示)。

**每个K/V服务器每隔PingInterval时间发送一次Ping RPC**。viewservice用当前的view响应Ping。Ping操作让viewservice知道K/V服务器的存在；告诉K/V当前的view值；告诉viewservice服务器K/V服务器当前知道的最新的view值。如果viewservice在DeadPings PingIntervals时间间隔内没有收到服务器的Ping，viewservice就认为这个服务器死掉了。当服务器在崩溃后重启，它应该发送一个或者多个参数为0的Pings，告知viewservice它宕机了(当然，重复的Ping(0)请求被解释为重复的宕机)。

**当(1)viewservice在DeadPings PingIntervals时间内没有收到主服务器或备份服务器的Ping，或者(2)主服务器或备份服务器宕机后重启，或者(3)没有备份服务器但是有可用的服务器时，就会产生一个新的view。**但是，viewservice不能改变view，直到主服务器从当前的view确认，它操作的确实是当前的view。如果viewservice还没有收到当前view值的主服务器的确认，viewservice就不会改变viewservice，即使它认为主服务器或者备份服务器已经死掉了。

这个确认规则防止了viewservice在K/V服务器之前获得多于1个的view。如果viewservice能够提前得到仲裁，那么，就需要一种更为复杂的设计来保存view值的历史，以便允许K/V查询以前的view值，然后在适当的时候删除以前的view值。这个确认规则的缺点是：如果主服务器在确认它收到的view之前宕机了，那么，viewservice不会改变view值，会永远停留在一个地方，并且不会继续执行。

提示：

1 你会想在server.go的ViewServer中添加一些域来跟踪viewservice中每个服务器接收到的Ping的最近时间。也许是个从服务器名字到time.Time的map。你能够通过time.Now()获取当前时间。

2 在ViewServer中添加域来跟踪当前的view值。

3 需要跟踪主服务器是否确认当前的view。

4 viewservice需要进行周期性的操作，比如，如果viewservice已经没有收到DeadPings个pings时，需要将备份服务器变成主服务器。这个这些代码添加在tick()中，它每隔PingInterval会调用一次。

5 **可能有不止2个服务器发送Ping。除了主备服务器之外的被当成备份服务器**。(译者：也就是说，viewservice要保存多个服务器的信息，而不只是主备服务器的信息，以便在备份服务器故障后，将额外的服务器提升为备份服务器)

6 **viewservice需要一种方式检测主服务器或者备份服务器宕机重启**。比如，主服务器可能宕机了，然后立刻重启，并没有中断Ping的发送。

## 3 测试场景

1 viewservice启动时，主备服务器都为空。此时，任何一个服务器向它Ping时，都被当作主服务器，并且viewnum自增。

2 如果某个服务器第一次向viewservice发送Ping时，但已经有主服务器，那么该服务器就会被当作备份服务器，并且viewnum自增。

3 当主服务器宕机时(即DeadPings PingInterval时间没有发送Ping)，备份服务器被提升为主服务器(当然，有备份服务器的情况下)。

4 当主服务器重启之后，它会变成备份服务器。

5 当主服务器故障，备份服务器会被提升为主服务器，如果还有可用的服务器时，就将可用的服务器提升为备份服务器。

6 主服务器宕机重启，但是，没有中断Ping的发送，它还是会被当作死掉了。

7 viewservice在开始下一个view时，会等待view的确认。

8 一个新的服务器不能作为主服务器。

## 4 实现

程序分为三个部分和一个测试文件。client.go:客户端代码。common.go:通信的参数和常量定义。server.go:服务端代码。

client.go:主要的函数有Ping(向viewserver发送Ping)，Get(返回当前的view)，Primary(返回当前的主服务器)。

server.go:主要的函数有Ping(接收客户端的Ping消息并进行处理)，Get(接收客户端的Get消息并返回当前的view)，tick(每隔PingInterval执行一次)。 

由于客户端要获取当前的view，因此，需要在viewserver保存下列变量：

``` GO
// type ViewServer struct
  curView uint // 当前的view值
  primary string // 当前的主服务器
  backup string // 当前的备份服务器
  counter map[string]int // 各个服务器的计数器
```

下面分别给出viewserver的Ping、Get、tick的代码：

``` GO
func (vs *ViewServer) Ping(args *PingArgs, reply *PingReply) error {

  // Your code here.
  client := args.Me
  viewNum := args.Viewnum

  if viewNum == 0 {
    if vs.primary == "" { // view值为0，并且主服务器为空串时，就设置主服务器(测试场景1)
      vs.curView++
      vs.primary = client
      vs.counter[vs.primary] = 0
    } else if vs.backup == "" { // 当备份服务器为空时，就设置备份服务器(测试场景2)
      vs.backup = client
      vs.counter[vs.backup] = 0
    } else {
      if vs.primary == client { // 主服务器故障重启，就将主备服务器互换(测试场景6)
        vs.counter[vs.backup], vs.counter[vs.primary] = vs.counter[vs.primary], vs.counter[vs.backup]
        vs.primary, vs.backup = vs.backup, vs.primary
        vs.counter[vs.primary] = 0
      } else {
        vs.counter[client] = 0
      }
    }
  } else if vs.primary == client && viewNum == vs.curView { // 主服务器对当前的view值进行确认
    vs.curView++
    vs.counter[client] = 0
  } else {
    vs.counter[client] = 0
  }

  reply.View.Viewnum = vs.curView
  reply.View.Primary = vs.primary
  reply.View.Backup = vs.backup

  return nil
}

// 
// server Get() RPC handler.
//
func (vs *ViewServer) Get(args *GetArgs, reply *GetReply) error {

  // Your code here.
  reply.View.Viewnum = vs.curView
  reply.View.Primary = vs.primary
  reply.View.Backup = vs.backup

  return nil
}


//
// tick() is called once per PingInterval; it should notice
// if servers have died or recovered, and change the view
// accordingly.
//
func (vs *ViewServer) tick() {
  // Your code here.
  vs.counter[vs.primary]++ // 对主备服务器的计数器递增
  vs.counter[vs.backup]++

  if vs.counter[vs.primary] > 5 { // 计数器大于5，说明主服务器故障了(测试场景3)
    delete(vs.counter, vs.primary) // 将主服务器从当前的计数器map中删除
    if vs.backup != "" { // 如果当前备份服务器不为空，就将备份服务器提升为主服务器
      pri := vs.primary
      vs.counter[vs.primary] = vs.counter[vs.backup]
      vs.primary = vs.backup
      vs.counter[vs.backup] = 0

      var k string
      for k, _ = range vs.counter {
        if k != pri && k != vs.backup {
          break
        }
      }

      // 当备份服务器提升为主服务器后，查找一个可以提升为备份服务器的服务器(测试场景5)
      if k != pri && k != vs.backup { 
        vs.backup = k
        vs.counter[vs.backup] = 0
      } else {
        vs.backup = ""
      }
    }
  }

  if vs.counter[vs.backup] > 5 { // 备份服务器故障了
    delete(vs.counter, vs.backup)
    var k string
    var v int
    for k, v = range vs.counter {
      if k != "" && k != vs.primary && k != vs.backup {
        break
      }
    }

    if k != "" && k != vs.primary && k != vs.backup { // 找一个可以提升为备份服务器的服务器
      vs.backup = k
      vs.counter[vs.backup] = v
    } else { // 没有可以提升为备份服务器的服务器时，就将备份服务器设置为空串
      vs.backup = ""
    }
  }
}
```

## 5 测试结果

![](https://github.com/luofengmacheng/distributed_system/blob/master/pic/viewservice1.png)

To be continued ...
