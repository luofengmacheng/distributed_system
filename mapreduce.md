# MapReduce

[《Distributed Computer Systems Engineering》](http://css.csail.mit.edu/6.824/2014/)是MIT的一门关于分布式系统开发的课程，这门课程讲到了分布式系统的许多方面。而且，这门课会每年更新，与时俱进，比如，之前的课程用的是C语言，现在用的是GO语言，不像国内的课程，可能十几年，甚至几十年还是原来那样。国内的课程很多都停留在理论阶段，让学生没有实践的机会，但是，这门课给了几个实验，让学生们在实验的过程中能够快速地学习。

MapReduce是Google抽象出来的一个分布式计算模型，虽然Google已经宣称不再使用MapReduce，但是，MapReduce在大规模分布式计算带来了很深远的影响。MapReduce并不是一个系统，而是Google从他们所处理的问题中抽象出来的一个模型，这个模型很像一个递归的思想：
将问题分为多个小问题，对这些小文件求解，然后将这些小问题的解进行合并，就得到了最终的解。
不熟悉MapReduce的可以看看Google的这篇论文：

实验1给出了一个简单的MapReduce的实现。

下面从代码的角度简单解释下：

首先是src/main/wc.go，这就是使用MapReduce计算单词个数的主函数所在的文件。
``` GO
// src/main/wc.go

// our simplified version of MapReduce does not supply a
// key to the Map function, as in the paper; only a value,
// which is a part of the input file contents
func Map(value string) *list.List {
}

// iterate over list and add values
func Reduce(key string, values *list.List) string {
}

// Can be run in 3 ways:
// 1) Sequential (e.g., go run wc.go master x.txt sequential)
// 2) Master (e.g., go run wc.go master x.txt localhost:7777)
// 3) Worker (e.g., go run wc.go worker localhost:7777 localhost:7778 &)
func main() {
  if len(os.Args) != 4 {
    fmt.Printf("%s: see usage comments in file\n", os.Args[0])
  } else if os.Args[1] == "master" {
    if os.Args[3] == "sequential" {
      mapreduce.RunSingle(5, 3, os.Args[2], Map, Reduce)
    } else {
      mr := mapreduce.MakeMapReduce(5, 3, os.Args[2], os.Args[3])
      // Wait until MR is done
      <- mr.DoneChannel
    }
  } else {
    mapreduce.RunWorker(os.Args[2], os.Args[3], Map, Reduce, 100)
  }
}
```

这里面包含三个函数：main、Map、Reduce。
根据main函数上面的解释知道，有三种方式可以运行main函数：
1 顺序执行(go run wc.go master x.txt sequential) 也就是在一台机器上执行，不采用分布式
2 Master(go run wc.go master x.txt localhost:7777) 以Master身份执行
3 Worker(go run wc.go worker localhost:7777 localhost:7778 &)以Worker身份执行
Map和Reduce也就是我们的实验1的第一部分。
因此，我们首先可以看看如何采用顺序执行：
``` GO
mapreduce.RunSingle(5, 3, os.Args[2], Map, Reduce)
```
这个函数调用mapreduce这个包中的RunSingle函数。

mapreduce包中RunSingle代码如下：
``` GO
func RunSingle(nMap int, nReduce int, file string,
               Map func(string) *list.List,
               Reduce func(string,*list.List) string) {
  mr := InitMapReduce(nMap, nReduce, file, "")
  mr.Split(mr.file)
  for i := 0; i < nMap; i++ {
    DoMap(i, mr.file, mr.nReduce, Map)
  }
  for i := 0; i < mr.nReduce; i++ {
    DoReduce(i, mr.file, mr.nMap, Reduce)
  }
  mr.Merge()
}
```

这个函数的代码很短，通过这个函数就知道了MapReduce的三个操作：map、reduce、merge。
1 InitMapReduce，对mr进行初始化，mr是一个结构体指针，其中包含了一些重要信息。
2 Split，对文件按照map的个数进行分割，每个map操作对应一个文件。
3 DoMap，对Split的每个文件进行map操作。
4 DoReduce，对map的结果文件进行reduce操作。
5 Merge，对reduce的结果进行合并。

## 1 Split文件分割

map处理的是一个文件，为了让每个map处理一个文件，先对初始给定的一个文件按照map的个数进行分割。

假设给定的文件是a.txt，文件大小为1000个字节，map的个数是3。
那么，Split会将这个文件分成3个文件，依次为mrtmp.a.txt-0，大小为334个字节；mrtmp.a.txt-1，大小为334个字节；mrtmp.a.txt-2，大小为332字节。

## 2 DoMap

对Split的结果文件执行map操作，每个分割的文件对应一个map操作。

``` GO
func DoMap(JobNumber int, fileName string,
           nreduce int, Map func(string) *list.List) {
  name := MapName(fileName, JobNumber) // 根据文件名和作业号就可以知道这次map操作处理的文件名，比如文件名为a.txt，作业号为1，那么要处理的文件名为mrtmp.a.txt-1
  file, err := os.Open(name)
  if err != nil {
    log.Fatal("DoMap: ", err);
  }
  fi, err := file.Stat();
  if err != nil {
    log.Fatal("DoMap: ", err);
  }
  size := fi.Size()
  fmt.Printf("DoMap: read split %s %d\n", name, size)
  b := make([]byte, size);
  _, err = file.Read(b); // 打开mrtmp.a.txt-1，根据文件大小分配存储空间，然后将文件内容读取到所分配的存储空间中。
  if err != nil {
    log.Fatal("DoMap: ", err);
  }
  file.Close()
  res := Map(string(b)) // 对这个文件的内容执行用户定义的Map操作
  // XXX a bit inefficient. could open r files and run over list once
  for r := 0; r < nreduce; r++ { // 对每个文件创建reduce个文件，假设有2个reduce，文件名依次为mrtmp.a.txt-1-0,mrtmp.a.txt-1-1。
    file, err = os.Create(ReduceName(fileName, JobNumber, r))
    if err != nil {
      log.Fatal("DoMap: create ", err);
    }
    enc := json.NewEncoder(file)
    for e := res.Front(); e != nil; e = e.Next() { // 遍历Map操作返回的链表，链表的内容形如{["abc", "1"], ["def", "1"], ["abc", "1"]}
      kv := e.Value.(KeyValue) // 获取链表遍历的当前值
      if hash(kv.Key) % uint32(nreduce) == uint32(r) { // 对键进行哈希，放到它应该放到的文件中，也就是说，map的结果将键相等的键值对放到了同一个文件中
        err := enc.Encode(&kv);
        if err != nil {
          log.Fatal("DoMap: marshall ", err);
        }
      }
    }
    file.Close()
  }
}
```

## 3 DoReduce

``` GO
func DoReduce(job int, fileName string, nmap int,
              Reduce func(string,*list.List) string) {
  kvs := make(map[string]*list.List) // 创建一个map，键的类型为string，值的类型是*list.List
  for i := 0; i < nmap; i++ {
    name := ReduceName(fileName, i, job)
    fmt.Printf("DoReduce: read %s\n", name)
    file, err := os.Open(name)
    if err != nil {
      log.Fatal("DoReduce: ", err);
    }
    dec := json.NewDecoder(file)
    for {
      var kv KeyValue
      err = dec.Decode(&kv);
      if err != nil {
        break;
      }
      _, ok := kvs[kv.Key]
      if !ok {
        kvs[kv.Key] = list.New()
      } 
      kvs[kv.Key].PushBack(kv.Value) // 读取执行reduce操作的文件，将键相同的键值对的值放到该键对应的链表中
    }
    file.Close()
  }
  var keys []string
  for k := range kvs { // keys获得所有的键
    keys = append(keys, k)
  }
  sort.Strings(keys)
  p := MergeName(fileName, job) // 为本次reduce创建一个文件，假设是第一个reduce操作，文件名为mrtmp.a.txt-res-0
  file, err := os.Create(p)
  if err != nil {
    log.Fatal("DoReduce: create ", err);
  }
  enc := json.NewEncoder(file)
  for _, k := range keys { // 遍历所有的键，对每个键以及它对应的链表执行Reduce操作，并将结果写入到mrtmp.a.txt-res-0中
    res := Reduce(k, kvs[k])
    enc.Encode(KeyValue{k, res})
  }
  file.Close()
}
```

## 4 Merge

``` GO
func (mr *MapReduce) Merge() {
  DPrintf("Merge phase")
  kvs := make(map[string]string)
  for i := 0; i < mr.nReduce; i++ { // 合并所有的DoReduce的结果文件
    p := MergeName(mr.file, i)
    fmt.Printf("Merge: read %s\n", p)
    file, err := os.Open(p)
    if err != nil {
      log.Fatal("Merge: ", err);
    }
    dec := json.NewDecoder(file)
    for {
      var kv KeyValue
      err = dec.Decode(&kv);
      if err != nil {
        break;
      }
      kvs[kv.Key] = kv.Value // 将所有的键值对存储到kvs中
    }
    file.Close()
  }
  var keys []string
  for k := range kvs { // keys获得所有的键
    keys = append(keys, k)
  }
  sort.Strings(keys)

  file, err := os.Create("mrtmp." + mr.file) // 创建最终的文件mrtmp.a.txt
  if err != nil {
    log.Fatal("Merge: create ", err);
  }
  w := bufio.NewWriter(file)
  for _, k := range keys { // 根据键将键的个数存储到文件中
    fmt.Fprintf(w, "%s: %s\n", k, kvs[k])
  }
  w.Flush()
  file.Close()
}
```

下面就是上面四个操作的结构图：
![]()

通过上面的解释可以知道用户的Map和Reduce要干什么：
* func Map(value string) *list.List 参数的类型是string，它包含了这次处理的文本的内容，返回值的类型是*list.List，它是所有的单词的键值对。因此，Map就是获得alue中的单词，然后将{单词，"1"}存储到一个链表中。
* func Reduce(key string, values *list.List) string 参数有两个，一个是key，它表示这次reduce对应的单词，values是一个链表，它包含了key出现的次数的一个链表，返回值的类型是string，它就是key出现的个数。因此，Reduce就是对values这个链表中的值进行累加。

于是有：
``` GO
func GetWords(ch rune) bool {
  if ch == ' ' || ch == '\t' || ch == '\n' || ch == '\r' || ch == '\v' || ch == '\f'{
    return true;
  }

  return false;
}

func Map(value string) *list.List {
  lst := list.New()
  words := strings.FieldsFunc(value, GetWords) // 用FieldsFunc对value进行分割，得到的就是一个单词的切片，它的第一个参数就是要分割的字符串，第二个参数就是分割的条件

  for _, word := range words {
    lst.PushBack(mapreduce.KeyValue{string(word), "1"})
  }

  return lst
}

// iterate over list and add values
func Reduce(key string, values *list.List) string {
  cnt := 0
  for e := values.Front(); e != nil; e = e.Next() {
    i, _ := strconv.Atoi(e.Value.(string))
    cnt += i;
  }

  return strconv.Itoa(cnt)
}
```
