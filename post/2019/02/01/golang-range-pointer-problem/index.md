# Golang中range指针数据的坑


在Golang中使用`for range`语句进行迭代非常的便捷，但在涉及到指针时就得小心一点了。

下面的代码中定义了一个元素类型为`*int`的通道`ch`：

```go
package main

import (
	"fmt"
)

func main() {
	ch := make(chan *int, 5)
	
    //sender
    input := []int{1,2,3,4,5}
	
    go func(){
		for _, v := range input {
			ch <- &v
		}
		close(ch)
	}()
	//receiver
	for v := range ch {
		fmt.Println(*v)
	}
}
```

在上面代码中，发送方将`input`数组发送给`ch`通道，接收方再从`ch`通道中接收数据，程序的预期输出应该是：

```bash
1
2
3
4
5
```

现在运行一下程序，得到的输出如下：

```bash
5
5
5
5
5
```

很明显，程序并没有达到预期的结果，那么问题出在哪里呢？我们将代码稍作修改：

```go
//receiver
for v := range ch {
	fmt.Println(v)
}
```

得到如下输出：

```bash
0x416020
0x416020
0x416020
0x416020
0x416020
```

可以看到，5次输出变量`v`（`*int`）都指向了同一个地址，返回去检查一下发送部分代码：

```go
for _, v := range input {
	ch <- &v
}
```

问题正是出在这里，在`for range`语句中，`v`变量用于保存迭代`input`数组所得的值，但是`v`只被声明了一次，此后都是将迭代`input`出的值赋值给`v`，`v`变量的内存地址始终未变，这样再将`v`的地址发送给`ch`通道，发送的都是同一个地址，当然无法达到预期效果。

解决方案是，引入一个中间变量，每次迭代都重新声明一个变量`temp`，赋值后再将其地址发送给`ch`： 

```go
for _, v := range input {
	temp := v
	ch <- &temp
}
```

抑或直接引用数据的内存（推荐，无需开辟新的内存空间）：

```go
for k, _ := range input {
    c <- &input[k]
}
```

再次运行，就可看到预期的效果。以上方案是用于讨论`range`语句带来的问题，当然，平时还是尽量避免使用指针类型元素的通道。
