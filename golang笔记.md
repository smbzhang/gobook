# golang 学习踩坑笔记

https://www.kancloud.cn/uvohp5na133/golang/933997

## 数据结构

### 引用类型的数据结构

#### new 和 make 的用法
golang的引用类型包括slice，map和channel。它们有复杂的内部结构，除了申请内存外，还需要初始化相关属性。

内置的函数new计算类型大小，为其分配零值内存，返回指针。而make会被编译器翻译成具体的创建函数，由其分配内存并且初始化成员结构，返回对象而非指针。

**new和make的区别**
```
make 用来创建map、slice、channel
new 用来创建值类型
new 和 make 均是用于分配内存：
new 用于值类型和用户定义的类型，如自定义结构，make 用于内置引用类型（切片、map 和管道）。
它们的用法就像是函数，但是将类型作为参数：new(type)、make(type)。new(T) 分配类型 T 的零值并返回其地址，
也就是指向类型 T 的指针。它也可以被用于基本类型：v := new(int)。
make(T) 返回类型 T 的初始化之后的值，因此它比 new 进行更多的工作。new() 是一个函数，不要忘记它的括号。
```
#### slice 切片

切片是引用类型

```
func main() {
	a := []int{0, 0, 0} // 提供初始化表达式。
	a[1] = 10

	b := make([]int, 3) // make slice
	b[1] = 10

	c := new([]int)
    // 切片的指针就相当于是一个二重指针，二重指针是不能进行[]操作的
	c[1] = 10 // ./main.go:11:3: invalid operation: c[1] (type *[]int does not support indexing)

    // 二重指针的正确使用方式
    s := new([]int32)
	*s = append((*s), 4)
	(*s)[0] = 5
}
```

##### slice结构的注意事项

**切片作为函数参数进行传递的时候, 只有修改才会影响外部，append追加的方式可能无法对外部造成影响**

```
func sliceParam(tempSlice []int){
    // 这里如果tempSlice的底层数组没有足够大就会重新分配底层数组，这样就不会影响到函数外部的slice
	tempSlice = append(tempSlice, 4,5,6)
}
func sliceParamPointer(tempSlicePointer *[]int){
	*tempSlicePointer = append(*tempSlicePointer,4,5,6)
}
func main() {
	slice := []int{1,2,3}
	sliceParam(slice)
	fmt.Println(slice) //[1,2,3]
	sliceParamPointer(&slice)
	fmt.Println(slice) //[1,2,3,4,5,6]
}
```
究其原因也很明朗，其实就是函数传参永远都是值传递！
1. 当切片作为参数的时候，一旦对tempSlice追加数据。那么tempSlice变量的值，
即所保存的内存地址就会变化。
换句话说此时追加数据操作是针对tempSlice变量变化之后的值对应的内存地址
但是由于值传递，slice只是把之前的切片地址作为值传到了函数内的tempSlice上
因此tempSlice的值做出变更不会对slice变量有任何影响，
因此切片作为参数的时候，一旦追加，就无法对外部造成影响。
2. 当切片指针作为参数的时候，必须清楚的是，仍然是值传递！！！
只不过传到tempSlicePointer上的是slice的指针
此时通过\*tempSlicePointer进行追加操作时，同样\*tempSlicePointer的值也会发生变化
关键的来了
那就是tempSlicePointer因为是二重指针，
所以*tempSlicePointer表示的是原slice切片的地址
所以即便*tempSlicePointer的值发生怎样的变化，都相当于原slice切片的地址发生变化
而既然函数内通过*tempSlicePointer对切片的追加，都相当于对原slice切片进行追加
所以，通过切片指针在函数内对切片的追加操作就能够对函数外的切片产生影响。

## 运算符优先级

- *指针单目运算符的优先级小于 []这个单目运算符

```
var arr [5]int = [5]int{1,2,3,4,5};
p := &arr;
fmt.Println(*p[0]);
```
上面就是一个错误的用法，p是数组指针，p[0]先被解析，内存访问失败

- 指针

golang 对于一重指针是可以不用写 * 来进行反引的，但是对于二重指针就不行了，比如对slice进行取地址操作

sclie 切片数据结构本身就是一个引用类型，再对其进行取地址那么就是二重指针。就必须使用 *来进行反指针。

```
p := &slice;
fmt.Printf("%p\n", p);//0xc000012446;
fmt.Println(p);            //0xc0000aa362     相当于传统c中的*p 
fmt.Println(*p);        //0xc0000ac141    相当于传统c中的**p
fmt.Println(p[0]);        //违法            相当于传统c中的(*p)[0]
fmt.Println((*p)[0]);    //1                相当于传统c中的(**p)[0]
```

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

### golang的nil是什么

golang的nil标识的是无,或者零值. 在Go语言中，如果你声明了一个变量但是没有对它进行赋值操作，那么这个变量就会有一个类型的默认零值。这是每种类型对应的零值：

```
bool       -> false                              
numbers    -> 0                                 
string     -> ""      

pointers   -> nil
slices     -> nil
maps       -> nil
channels   -> nil
functions  -> nil
interfaces -> nil
```
