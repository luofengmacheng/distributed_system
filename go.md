go

1 变量

var a int
a = 5

a := 5

2 string字符串是不能修改的

3 defer 在defer后指定的函数会在函数退出前调用

4 go中的内存分配

new返回指针，如p := new(int)，并且内存空间为0值，可以针对任何类型
make返回的是引用，如s := make([]int)，并且内存空间有初始值(非0)，只能创建slice,map,channel，这是因为这三种类型的引用在使用前必须被初始化。

5 函数可以返回局部变量的地址，函数返回后，相关的存储区域仍然存在

6 func (in1 *NameAge) doSomething(in2 int) { }
var n *NameAge
n.doSomething(5)

也可以这样调用：
var n NameAge
n.doSomething(5)

原因：如果x可以获取地址，并且&x的方法中包含了m，x.m()是(&x).m()更短的写法。

7 go中类似C++中template的方法

7.1 定义一个有着若干排序相关的方法的接口类型。至少需要获取slice长度的函数，比较两个值的函数和交换函数。

type Sorter interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

7.2 定义用于排序slice的新类型

type Xi []int
type Xs []string

7.3 实现Sorter接口的方法

整数
func (p Xi) Len() int {
	return len(p)
}

func (p Xi) Less(i int, j int) bool {
	return p[j] < p[i]
}

func (p Xi) Swap(i int, j int) {
	p[i], p[j] = p[j], p[i]
}

字符串
func (p Xs) Len() int {
	return len(p)
}

func (p Xs) Less(i int, j int) bool {
	return p[j] < p[i]
}

func (p Xs) Swap(i int, j int) {
	p[i], p[j] = p[j], p[i]
}

7.4 函数实现
func Sort(x Sorter) {
	for i := 0; i < x.Len() - 1; i++ { 1
		for j := i + 1; j < x.Len(); j++ {
			if x.Less(i, j) {
				x.Swap(i, j)
			}
		}
	}
}
