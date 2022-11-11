```
64-bit value: high 32 bits are counter, low 32 bits are waiter count.
64-bit atomic operations require 64-bit alignment, but 32-bit compilers only guarantee that 64-bit fields are 32-bit aligned.
For this reason on 32 bit architectures we need to check in state() if state1 is aligned or not, and dynamically "swap" the field order if needed.
```

```
64位的值 分两段
	高32位是计数值
	低32位是waiter的计数
64位值的原子操作需要64位对其，但32位编译器仅保证64位字段是32位对齐的
出于这个原因，在32位架构上，我们需要通过state()检查state1是否对齐，不对齐时，需要交换字段顺序
```



问题：

计数值 和 waiter计数



```go
type WaitGroup struct {
	noCopy noCopy
    
    state1 uint64
	state2 uint32
}
```

- noCopy
  - 避免复制
  - 通过vet工具可以检验