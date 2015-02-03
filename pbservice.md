# 实验2任务2：primary/backup key/value service

## 1 实验介绍

K/V服务应该可以在分区的情况下也操作正确：服务器在出现临时网络故障时不会崩溃，或者可以同某些计算机通信。

操作正确意味着：调用Clerk.Get(k)返回成功调用Clerk.Put(k, v)或者Clerk.PutHash(k, v)的最后的值，或者当这个键没有进行Put()时，就返回空串。所有的操作都必须提供最多一次的语义。


