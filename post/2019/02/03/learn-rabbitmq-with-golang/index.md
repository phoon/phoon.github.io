# RabbitMQ初涉以及Golang实践


在并发编程中，多线程并发协作时采用生产者消费者模式是一个良好的解决方案。生产者线程将生成的数据放入一个阻塞队列中，消费者则直接从该队列中获取数据，这样做的目的是为了降低生产者与消费者之间的耦合性，同时也平衡了两者的不对等的处理能力。

为了达到上述目的，还可以考虑采用消息中间件。由此也引出我们今天的主题：`RabbitMQ`。`RabbitMQ`实现了`AMQP` `(Advanced Message Queue Protocol)`协议，是目前广泛使用的消息中间件之一。

`RabbitMQ`中有几个重要的概念：

- `Queue`： 消息队列，是`RabbitMQ`的核心
- `Binding`： 绑定，`Exchange`通过与`Queue`绑定并为每个`Queue`设置`Routing Key`从而达到路由功能

- `Channel`： 消息通道，每一个`Channel`代表一个客户端会话任务

- `Exchange`： 交换器，制定消息传递的规则，选择路由

- `Routing Key：` 路由关键字，`Exchange`根据此项选择投递消息到哪个`Queue`

- `Virtual Host`： 虚拟机，用于隔离用户权限

  

#### 安装RabbitMQ

`Archlinux`:

```bash
sudo pacman -S rabbitmq
```

安装完成之后，启动`RabbitMQ`：

```bash
sudo systemctl start rabbitmq.service
```

此时，`RabbitMQ`就开始运行了，默认只采用`AMQP`协议。如果想使用网页来管理服务器，可以激活对应的插件：

```bash
sudo rabbitmq-lugins enable rabbitmq_management
```

`AMQP`协议的端口为`5672`，网页管理台端口为`15672`，默认用户名和密码均为`guest`(记得改密码！)

再将`RabbitMQ`重启一下：

```bash
sudo systemctl restart rabbitmq.service
```

新建用户和虚拟机：

```
//新建一个用户
sudo rabbitmqctl add_user username password 
//新建一个虚拟机
sudo rabbitmqctl add_vhost NewHost
//设置用户角色
sudo rabbitmqctl set_user_tags username monitoring
//设置 /NewHost对于用户username可用
sudo rabbitmqctl set_permissions -p NewHost username ".*" ".*" ".*"
```

#### Golang使用RabbitMQ

想要通过`Golang`来使用`RabbitMQ（AMQP协议版）`,需要下载`AMQP`库：

```bash
go get github.com/streadway/amqp
```

然后编写`producer.go`与`consumer.go`两个程序：

```go
/*
	producer.go
	Author: Peven
*/
package main

import (
	"log"

	"github.com/streadway/amqp"
)

func main() {
	//start a new amqp connection
	conn, err := amqp.Dial("amqp://username:password@localhost:5672/NewHost")
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	//declare a channel
	ch, err := conn.Channel()
	if err != nil {
		log.Fatal(err)
	}
	defer ch.Close()

	//declare a queue
	queue, err := ch.QueueDeclare(
		"TestQueue", //queue name
		true,        //durable
		false,       //auto-delete when unused
		false,       //exclusive
		false,       //no-wait
		nil,         //args
	)
	if err != nil {
		log.Fatal(err)
	}

	//declare a exchange
	err = ch.ExchangeDeclare(
		"TestExchange", //exchange name
		"direct",       //type
		true,           //durable
		false,          //auto-delete when unused
		false,          //internal
		false,          //no-wait
		nil,            //args
	)
	if err != nil {
		log.Fatal(err)
	}

	//binding a queue
	err = ch.QueueBind(
		queue.Name,     //queue name
		"routing_key",  //routing key
		"TestExchange", //exhchange name
		false,          //no-wait
		nil,            //args
	)
	if err != nil {
		log.Fatal("1:", err)
	}

	//publish a message
	err = ch.Publish(
		"TestExchange", //exchange name
		"routing_key",  //routing key
		false,          //mandatory
		false,          //immediate
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte("This is a message"),
		},
	)
	if err != nil {
		log.Fatal(err)
	}
}

```

```go
/*
	consumer.go
	Author: Peven
*/
package main

import (
	"fmt"
	"log"
	"sync"

	"github.com/streadway/amqp"
)

func main() {
    //start a new amqp connection
	conn, err := amqp.Dial("amqp://username:password@localhost:5672/NewHost")
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	
    //declare a channel
	ch, err := conn.Channel()
	if err != nil {
		log.Fatal(err)
	}
	defer ch.Close()
	
    //declare a queue
	queue, err := ch.QueueDeclare(
		"TestQueue", //queue name
		true,        //durable
		false,       //auto-delete when unused
		false,       //exclusive
		false,       //no-wait
		nil,         //args
	)
	if err != nil {
		log.Fatal(err)
	}
	
    //get the delivery results
	msgs, err := ch.Consume(
		queue.Name, // queue name
		"peven",    // consumer name
		true,       // auto-ack
		false,      // exclusive
		false,      // no-local
		false,      // no-wait
		nil,        // args
	)
	
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		for m := range msgs {
			fmt.Println("Receive a message: ", string(m.Body))
		}
	}()
	wg.Done()
}

```

 运行完`producer`之后，打开终端输入以下命令查看`Exchange`：

```bash
sudo rabbitmqctl list_exchanges -p NewHost
```

输出如下：

```bash
Listing exchanges for vhost NewHost ...
name    		type
amq.topic       	topic
amq.direct      	direct
TestExchange    	direct	//新创建的交换器
amq.fanout      	fanout
amq.rabbitmq.trace  	topic
amq.match       	headers
amq.headers     	headers
        		direct	//默认路由，""
```

输入`sudo rabbitctl list_bindings -p NewHost `查看建立的绑定：

```bash
$ sudo rabbitmqctl list_bindings -p NewHost --no-table-headers

Listing bindings for vhost NewHost...
        exchange        TestQueue       queue   TestQueue       []
TestExchange    exchange        TestQueue       queue   routing_key     []

```

同时，运行`consumer`也打印出了`producer`发送的消息。
