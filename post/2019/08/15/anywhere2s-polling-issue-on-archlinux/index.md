# 解决Archlinux下蓝牙鼠标滞后感严重的问题


## 契机

最近入手了`Logitech MX Anywhere 2S`鼠标，欣喜的在Arch中连接上蓝牙一试，瞬间傻眼...

移动鼠标，指针掉帧严重，有很重的滞后感，运行`evhz`一看，平均`polling rate`只有`22Hz`, 这怎么玩嘛。

这里解释一下鼠标的`polling rate`：`polling rate`就是鼠标的轮询（刷新）率， 是鼠标向计算机报告其位置的频率，以Hz为单位。那么这里的`22Hz`意味着什么呢？当前鼠标每秒钟向计算机报告位置22次，也就是大约每`45.5ms`报告一次，不卡就怪了。

## 解决方法

在各处论坛爬楼后，给出以下两种解决方案：

> 此处所示命令均为bash风格

### 通过debugfs（内核调试）

运行以下命令：

```bash
$ sudo echo 0 > /sys/kernel/debug/bluetooth/hci0/conn_latency
$ sudo echo 6 > /sys/kernel/debug/bluetooth/hci0/conn_min_interval
$ sudo echo 7 > /sys/kernel/debug/bluetooth/hci0/conn_max_interval
```

> 路径中的hci0表示你电脑蓝牙接收器的名称

### 借助hcitool工具

`hcitool`在`Archlinux`上已经弃用，不过你可以通过`AUR`来进行安装：

```bash
$ yay -S bluez-hcitool
```

然后运行：

```bash
$ export MOUSEHANDLE=`hcitool con | grep "XX:XX:XX:XX:XX:XX" | awk '{print $5}'`
$ sudo hcitool lecup --handle $MOUSEHANDLE --min 6 --max 7 --latency 0
```

> `XX:XX:XX:XX:XX:XX`替换为鼠标地址

## 总结

​	在实施以上两种方案后，鼠标滞后感消失，在`evhz`下的`polling rate`值也上升到正常水平`125Hz`左右。不过使用第一种方案的话，目前不知道会不会出现意想不到的后果。而且经我实际体验，重启机器后即失效，并需要在再次运行命令之后删除设备然后重新配对连接之后才能再次生效；第二种方案，每次设备断线重连即失效，好处是不用删除设备重新配对连接。权衡之下，第二种解决方案还是更胜一筹，虽然这些都只是权宜之计罢了。


