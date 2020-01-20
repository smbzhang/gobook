# golang 学习踩坑笔记

## 数据结构

### 引用类型的数据结构


## 运算符优先级

- *这个单目运算符的优先级小于 []这个单目运算符


## golang的一些特殊语法

### 切片相关的数据结构

#### ... 三个点的用法

```
// 可以接收任意个 string 的参数
func test(args ...string) {
	for _, v := range args {
		fmt.Println(v)
	}
}
func main() {
	var strs []string = []string{
		"smb",
		"bob",
		"alice",
	}
	// ... 切片被打散传入
	test(strs...)
	strs = test2()
	test(strs...)
}
func test2() []string {
	var strs1 []string = []string {
		"qqq",
		"www",
	}
	var strs2 []string = []string {
		"eee",
		"rrr",
	}
	//strs2 的元素被打散一个个append进 strs1 简洁了代码
	strs1 = append(strs1, strs2...)
	return strs1
}
```
从上面的这个例子中我们可以看到...的作用主要是两个

- 函数的参数可以通过...指定多个同类型的传入参数
- 用来打散切片成一个一个的切片元素，简化一个for的编程
