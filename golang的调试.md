# golang的调试方法记录

## pprof 性能测试工具的使用

https://zhuanlan.zhihu.com/p/51559344

PProf 关注的模块
```
CPU profile：报告程序的 CPU 使用情况，按照一定频率去采集应用程序在 CPU 和寄存器上面的数据
Memory Profile（Heap Profile）：报告程序的内存使用情况
Block Profiling：报告 goroutines 不在运行状态的情况，可以用来分析和查找死锁等性能瓶颈
Goroutine Profiling：报告 goroutines 的使用情况，有哪些 goroutine，它们的调用关系是怎样的
```

引入方式
```
import "net/http/pprof"
如果使用了默认的 http.DefaultServeMux（通常是代码直接使用 http.ListenAndServe("0.0.0.0:8000", nil)），只需要添加一行：
import _ "net/http/pprof"
如果应用使用了自定义的 Mux，则需要手动注册一些路由规则：
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```
