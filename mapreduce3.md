# 处理worker故障

## 1 介绍

前面的部分都没有处理worker故障的情况，实验1任务3就要求处理worker出现故障的情况。这里的故障，不一定就是worker由于故障而卡机了，也有可能是由于网络的问题，总之，就是Master突然连不上Worker了。

## 2 实验1任务3：处理worker故障

实验1任务2不需要考虑worker的故障，也就是说，将任务交给worker就行了，它一定会收到任务，并且完成。但是，现在必须考虑worker的故障，也就是说，有可能要将工作发送给worker，但是，worker实际上已经死掉了，因此，就必须将这个任务交给其它的worker。

### 2.1 One Failure

先来考虑一个worker死掉的情况。实验1任务2不用考虑worker故障，因此，找到一个没有工作的worker时，就将这个工作交给它，之后就可以分配其它的工作，所以，是个异步的模型。但是，如果要考虑worker死掉，方便的办法就是用同步模型，将工作交给一个worker时，等待它完成，如果worker执行工作失败了，就继续查找可用的worker。

因此，对代码进行了以下更改：

1 将WorkerInfo中的inUse bool改为state int，其中0代表这个worker没有执行工作，1代表这个worker正在执行工作，2代表这个worker故障了。

2 将assignJob的返回值修改为bool，并且在调用worker执行工作时，如果失败了，worker的state修改为2，然后返回false，如果成功了，worker的state修改为0，然后返回true。

``` GO
ok := call(worker.address, "Worker.DoJob", jobargs, &jobreply)
if ok == false {
  fmt.Printf("call Worker.DoJob %s error\n", job)
  worker.state = 2
  log.Printf("this work failed")
  return false
} else {
  worker.state = 0;
  lock.Lock()
  *comJob++
  lock.Unlock()
  return true
}
```

3 修改RunMaster中处理工作的代码，这里只给出处理map工作的代码，处理reduce工作的代码基本类似：

``` GO
  for {
    if workId >= mr.nMap {
      log.Printf("send %d jobs to workers", workId)
      break
    }

    wkinfo = nil
    for _, wkinfo = range mr.Workers {
      if wkinfo.state == 0 {
        break
      }
    }

    if wkinfo != nil && wkinfo.state == 0 {
      wkinfo.state = 1
      if mr.assignJob(wkinfo, Map, workId, comLock, &comJob) == true {
        workId++
      }
    }
  }
```

其实，主要的修改就在处理assignJob的返回值那里。

### 2.2 One Failure测试结果

![](https://github.com/luofengmacheng/distributed_system/blob/master/pic/mapreduce3.png)

### 2.3 Many Failure

要处理多个故障，就必须处理Master运行过程中，worker向Master注册自己。因此，必须修改之前的RunMaster中开始等待worker的两段代码。

因此，对代码进行了以下修改：

1 修改了RunMaster中等待worker注册的两段代码，用一个goroutine等待worker的注册：

``` GO
  go func() {
    for {
      workerAddr := <- mr.registerChannel
      _, ok := mr.Workers[workerAddr]
      if !ok {
        wkinfo := new(WorkerInfo)
        wkinfo.address = workerAddr
        wkinfo.state = 0
        mr.Workers[workerAddr] = wkinfo
      } else {
        mr.Workers[workerAddr].state = 0
        mr.Workers[workerAddr].address = workerAddr
      }
    }
  }()
```

2 在RunMaster中处理工作的循环中加了一句fmt.Printf()：

``` GO
  for {
    if workId >= mr.nMap {
      log.Printf("send %d jobs to workers", workId)
      break
    }

    wkinfo = nil
    for _, wkinfo = range mr.Workers {
      if wkinfo.state == 0 {
        break
      }
    }

    fmt.Printf("hello\n") // 加了这一句

    if wkinfo != nil && wkinfo.state == 0 {
      wkinfo.state = 1
      if mr.assignJob(wkinfo, Map, workId, comLock, &comJob) == true {
        workId++
      }
    }
  }
```

至于为什么要加这一句呢？我个人也没有完全搞明白，不过，经过我测试，发现了一些有趣的现象：

不加fmt.Printf()，然后在上面等待worker注册的goroutine中，在读取registerChannel值之后，加上一句打印语句，结果就是这样：

![](https://github.com/luofengmacheng/distributed_system/blob/master/pic/mapreduce4.png)

也就是说，根本就没有worker注册，当然，更没有工作被执行。

加了fmt.Printf()之后，就可以收到worker的注册。

因此，我推测，对工作进行处理for循环阻碍了等待用户注册的goroutine中的for循环的执行。这跟实验1任务2中添加time.Sleep()的道理类似。这涉及到goroutine的实现，一般的书籍都说，goroutine并不是真正的并行，而是并发，也就是说，同一时刻，只能有一个执行，于是，这里或许产生了饥饿现象才导致goroutine中的for循环没有接收worker的注册。

### 2.4 Many Failures测试

![](https://github.com/luofengmacheng/distributed_system/blob/master/pic/mapreduce5.png)
