# golang 内存泄露

## top 命令中关于内存的指标的含义
```
%MEM：Memory usage (RES) 内存占用
使用的物理内存

VIRT：Virtual Image (kb) 虚拟镜像
总虚拟内存的使用数量

SWAP：Swapped size (kb)
非驻留但是存在于程序中的内存，虚拟内存减去物理内存

RES：Resident size (kb)
非swap的物理内存

SHR：Shared Mem size (kb)
程序使用的共享内存，可以被其它进程所共享
```

## 常见场景

golang 的内存管理是golang的runtime自带的gc来进行内存管理的,gc会对unused mem 进行回收,但是如果我们程序中对堆对象没有几时的关闭和释放，就会被gc认为该内存还不能被释放。

具体的内存泄露的实例可以参照这个文章：

https://colobu.com/2019/08/28/go-memory-leak-i-dont-think-so/

### 场景一：substrings 引起的内存泄露 (获取长字符串中的一段导致长字符串未释放)

```
var s0 string // a package-level variable
// A demo purpose function.
func f(s1 string) {
	s0 = s1[:50]
	// Now, s0 shares the same underlying memory block
	// with s1. Although s1 is not alive now, but s0
	// is still alive, so the memory block they share
	// couldn't be collected, though there are only 50
	// bytes used in the block and all other bytes in
	// the block become unavailable.
}
func demo() {
	s := createStringWithLengthOnHeap(1 << 20) // 1M bytes
	f(s)
}
```

### 场景二：subslices 引起的内存泄露 (获取长slice中的一段导致长slice未释放)

```
var s0 []int

func g(s1 []int) {
	// Assume the length of s1 is much larger than 30.
	s0 = s1[len(s1)-30:]
}
Similarly to substrings, subslices may also cause kind-of memory leaking. In the following code, after the g function is called, 
most memory occupied by the memory block hosting the elements of s1 will be lost (if no more values reference the memory block).
```

我们可以这样解决泄露的问题

```
func g(s1 []int) {
	s0 = append([]int(nil), s1[len(s1)-30:]...)
	// Now, the memory block hosting the elements
	// of s1 can be collected if no other values
	// are referencing the memory block.
}
```

### 场景三：切片中的指针没有被置为nil 引起的内存泄露 (在长slice新建slice导致泄漏)
```
func h() []*int {
	s := []*int{new(int), new(int), new(int), new(int)}
	// do something with s ...

	return s[1:3:3]
}
As long as the returned slice is still alive, it will prevent any elements of s from being collected, which in consequence prevents the two memory blocks allocated for the two int values referenced by the first and the last elements of s from being collected.

If we want to avoid such kind-of memory leaking, we must reset the pointers stored in the lost elements.
```

上面的代码会泄露 s切片的第一个和最后一个元素，我们可以这样解决。
```
func h() []*int {
	s := []*int{new(int), new(int), new(int), new(int)}
	// do something with s ...

	// Reset pointer values.
	s[0], s[len(s)-1] = nil, nil
	return s[1:3:3]
}
```

### 场景四：goroutine泄漏,hang死的goroutines

```
Sometimes, some goroutines in a Go program may stay in blocking state for ever. Such goroutines are called hanging goroutines. Go runtime will not kill hanging goroutines, so the resources allocated for (and the memory blocks referenced by) the hanging goroutines will never get garbage collected.

There are two reasons why Go runtime will not kill hanging goroutines. One is that sometimes it is hard for Go runtime to judge whether or not a blocking goroutine will be blocked for ever. The other is sometimes we deliberately make a goroutine hanging. For example, sometimes we may let the main goroutine of a Go program hang to avoid the program exiting.

We should avoid hanging goroutines which are caused by some logic mistakes in code design.
```

### 场景五： time.Ticker未关闭导致泄漏

```
When a time.Timer value is not used any more, it will be garbage collected after some time.
But this is not true for a time.Ticker value. We should stop a time.Ticker value when it is not used any more.
```

对于 time.Time的值，在不使用之后会被gc处理，但是 time.Ticker 并不会，需要手动的执行Close操作。

### 场景六： 不正确的使用Finalizer导致泄漏


### 场景七： Deferring Function Call导致泄漏

这个场景在解决 stable-proxy 这个bug的时候遇到过。看下面的例子

```
func writeManyFiles(files []File) error {
	for _, file := range files {
		f, err := os.Open(file.path)
		if err != nil {
			return err
		}
		defer f.Close()

		_, err = f.WriteString(file.content)
		if err != nil {
			return err
		}

		err = f.Sync()
		if err != nil {
			return err
		}
	}
	return nil
}
```
上面的例子中，直到所有的文件被处理完毕之后，也就是函数退出之后才会执行Close函数。试想一下如果是死循环的话，那么就永远得不到释放了。

stable proxy就是这个问题，死循环，数据库连接池得不到释放。

**解决办法**

可以使用匿名函数来进行处理
```
func writeManyFiles(files []File) error {
	for _, file := range files {
        // 通过下面的匿名函数，defer f.Close 就会在匿名函数退出的时候退出了
		if err := func() error {
			f, err := os.Open(file.path)
			if err != nil {
				return err
			}
			// The close method will be called at
			// the end of the current loop step.
			defer f.Close()

			_, err = f.WriteString(file.content)
			if err != nil {
				return err
			}

			return f.Sync()
		}(); err != nil {
			return err
		}
	}

	return nil
}
```

## Case 总结

golang 的版本 - 1.8 1.9 会导致内存泄露

```
1 package main
2
3 import (
4     "log"
5     "net/http"
6     _ "net/http/pprof"  //  初始化pprof
7 )
8
9 type Node struct {
10     next *Node
11     payload [64]byte
12 }
13
14 func main() {
15     go func() {
16         log.Println(http.ListenAndServe(":8989", nil))
17     }()
18     curr := new(Node)
19     for {
20         curr.next = new(Node)
21         curr = curr.next
22     }
23 }
```
按照理解 curr 指向的是当前的结点,那么在进行for循环的时候，cur前面的结点因为不会被引用所以会被gc收集，但是在golang12之前并没有被gc所回收，进程的内存呈现持续增长的状态直到所有的内存被吃完。

为什么会出现这种情况呢？通过逃逸分析发现，18行的 new(Node) 并没有被分配到 heap 上面。这是因为 golang 会对小内存进行逃逸分析，curr没有被外部引用，就算是通过new函数创建出来的也是可能被分配在栈上面的。

```
$ go tool compile -m main.go
main.go:15: func literal escapes to heap
main.go:15: func literal escapes to heap
main.go:20: new(Node) escapes to heap
main.go:16: http.ListenAndServe(":8989", nil) escapes to heap
main.go:18: main new(Node) does not escape
main.go:16: main.func1 ... argument does not escape
```
可以看到18行的new(Node) does not escape，并没有escape到heap上面去，而是在栈上面。在使用golang 12之前的版本会导致，栈上的curr直到main退出才会释放，curr又引用着下面的结点，导致下面在heap的结点也没办法释放，所以内存会无限增长。

如果我们这么改一下呢？
```
1	package main
2
3	import (
4		"log"
5		"net/http"
6		_ "net/http/pprof"  //  初始化pprof
7		"fmt"
8	)
9
10	type Node struct {
11		next *Node
12		payload [64]byte
13	}
14
15	func main() {
16		go func() {
17			log.Println(http.ListenAndServe(":8989", nil))
18		}()
19		curr := new(Node)
        // 这里打印一下
20		fmt.Println(curr)
21		for {
22			curr.next = new(Node)
23			curr = curr.next
24		}
25	}
```

再用逃逸工具分析一下：
```
$ go tool compile -m main.go
main.go:16: func literal escapes to heap
main.go:16: func literal escapes to heap
main.go:20: curr escapes to heap
main.go:19: new(Node) escapes to heap
main.go:22: new(Node) escapes to heap
main.go:17: http.ListenAndServe(":8989", nil) escapes to heap
main.go:20: main ... argument does not escape
main.go:17: main.func1 ... argument does not escape
```
发现curr逃逸到haep上去了，并且内存被正常的释放了。为什么呢？因为fmt.Println是这样定义的
```
func Println(a ...interface{}) (n int, err error) 
```
参数是一个interface，这是一个引用类型，被外部函数引用，逃逸到了heap上面不会被优化。也就能在运行中被正常的GC了，这样就不用担心内存泄露了。

**所以，一定要使用新版的golang，否则有很多坑**