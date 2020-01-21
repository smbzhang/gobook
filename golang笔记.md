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
<<<<<<< HEAD
fmt.Printf("%p\n", p);     //0xc000012446;
fmt.Println(p);            //0xc0000aa362      相当于传统c中的*p 
fmt.Println(*p);           //0xc0000ac141      相当于传统c中的**p
fmt.Println(p[0]);         //违法               相当于传统c中的(*p)[0]
fmt.Println((*p)[0]);      //1                 相当于传统c中的(**p)[0]
```

### 值类型（非引用类型）的数据结构

#### struct 结构体

struct建议使用指针进行传递，函数参数中有使用struct的建议使用指针，因为传递strcut应该就是要修改里面的值。
```
type S struct {
	names []string
	age int32
}
func test(s S)  {
	s.names[0] = "BOB"
}
func main() {
	var s S
	s.names = []string{
		"ALICE",
		"TOM",
	}
	test(s)
	fmt.Println(s) //{[BOB TOM] 0}  
}
```
上面你可能没有想改变struct里的值，但是因为struct里面有切片这个引用数据结构。所以还是被改变了。

## golang 语法

### for range
golang中for range语法非常方便，可以轻松的遍历array、slice、map等结构，但是它有一个特点，**就是会在遍历时把当前遍历到的元素，复制给内部变量**
```
p1 := People{
	age:  10,
	name: "smb",
	sex:  "fame",
}
p2 := People{
	age:  10,
	name: "smb",
	sex:  "fame",
}
p3 := People{
	age:  10,
	name: "smb",
	sex:  "fame",
}
people := []People {p1, p2, p3,}
for _, p := range people {
	p.age = 20
}
fmt.Println(p1.age)   // 输出 10
```
上面的程序并不会改变切片中的元素的值，因为for range 只会进行值拷贝，struct 又是非引用类型。

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
<<<<<<< HEAD

## golang 踩坑

### 使用ReadCloser
```
// ReadCloser is the interface that groups the basic Read and Close methods.
type ReadCloser interface {
	Reader
	Closer
}
for {
    str_num, err := reader.Read(buf)    
    if err != nil {
        if err == io.EOF || strings.Contains(err.Error(), "file already closed") {
            break
        }
    }
    if str_num > 0 {
        log_msg := buf[:str_num]
        master.WriteMsgIntoBosLogFile(task, string(log_msg))
    }

}
```
一般我们处理函数的返回的时候总是先判断是不是函数执行出错了，然后在进行返回数据的处理。但是ReadCloser这个接口的Read方法实现却不是这样的。
如果 Read 之后错误是 EOF 的话，返回的 num 是可能 > 0的，也就是说，buf中还是有数据的，如果想上面一样先判断err然后在读取buf，就总是会遗漏 EOF 的那个包的数据应该改成下面这样

```
for {
    str_num, err := reader.Read(buf)
    // 注意：str_num 的判断要放到 err 的判断之前:
    //     Callers should always process the n > 0 bytes returned before considering the error err.
    //     Doing so correctly handles I/O errors that happen after reading some bytes and also both of the allowed EOF behaviors.
    // Doc:
    //     https://golang.org/pkg/io/#Reader
    if str_num > 0 {
        log_msg := buf[:str_num]
        master.WriteMsgIntoBosLogFile(task, string(log_msg))
    }
    if err != nil {
        if err == io.EOF || strings.Contains(err.Error(), "file already closed") {
            break
        }
    }
}
```

### 数据库操作的值类型

因为golang没有null值，只有nil值，对于数据库字段是 NULL的，也就是在建表的过程中允许数据库的字段值为NULL，只能使用 sql.NullString sql.NullInt类型的值来进行读取rows的返回值。如果使用golang原生的string和int来读取字段允许为null的字段值，就会触发错误返回。

使用示例如下：
```
for rows.Next() {
	var task StatReverseBuildTask
	var ret_code int32
	var compareLogPath sql.NullString
	var logPath sql.NullString
	var failReason sql.NullString
	var exemptReason sql.NullString
	// init sql.NullString variable
	compareLogPath.String = ""
	logPath.String = ""
	failReason.String = ""
	exemptReason.String = ""
	err = rows.Scan(&task.CompareStatus,
					&compareLogPath,
					&task.Index,
					&task.ModuleName,
					&task.BranchName,
					&task.CommitID,
					&task.Status,
					&logPath,
					&ret_code,
					&failReason,
					&exemptReason)
	if err != nil {
		response := PostResponse{500, "Query data from database failed."}
		WriteJsonResponse(w, response)
		log.Println("Scan sql error: " + err.Error())
		return
	}
	if compareLogPath.Valid {
		task.CompareLogPath = compareLogPath.String
	}
	if logPath.Valid {
		task.LogPath = logPath.String
	}
	if failReason.Valid {
		task.FailReason = failReason.String
	}
	if exemptReason.Valid {
		task.ExemptReason = exemptReason.String
	}
```
