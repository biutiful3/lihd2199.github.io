

　　ifconfig这是linux很常用的一个命令，之前只知道它可以查询ip信息，也没有认真的研究过，近期看redis超时的问题，看到了一些相关东西，在网上找了些资料；

　　[Linux的ifconfig看到的信息详解 - 一颗桃子t - 博客园 (cnblogs.com)](https://www.cnblogs.com/taosiyu/p/11431758.html)

　　[从 ifconfig 读取网卡流量 - Linux - 大象笔记 (sunzhongwei.com)](https://www.sunzhongwei.com/read-the-card-from-the-ifconfig-traffic)

　　[Linux系统下ifconfig命令使用及结果分析 - OldHawk - 博客园 (cnblogs.com)](https://www.cnblogs.com/taobataoma/archive/2007/12/27/1016689.html)

```c
~$ ifconfig -a
eth0      Link encap:Ethernet  HWaddr 00:11:22:33:44:55  
          inet addr:10.10.10.10  Bcast:10.10.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:666433288 errors:0 dropped:2 overruns:0 frame:0
          TX packets:430278029 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:123925389232 (123.9 GB)  TX bytes:329876469836 (329.8 GB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:46060 errors:0 dropped:0 overruns:0 frame:0
          TX packets:46060 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:2303000 (2.3 MB)  TX bytes:2303000 (2.3 MB
```

上面是我的电脑的一个ifconfig命令，里面的IP地址和mac地址做了模糊。

首先可以看到是两个模块，第一个模块eth0是第一个网卡的意思，第二个是主机的回环地址，其实一般都是测试用的，也就是我们的127.0.0.1地址。

下面看一下第一个模块的内容：

第一行：连接类型：Ethernet（以太网）HWaddr（硬件mac地址）

第二行：网络地址：10.10.10.10    广播地址：10.10.255.255  子网掩码：255.255.0.0

第三行：UP（代表网卡开启状态）RUNNING（代表网卡的网线被接上）MULTICAST（支持组播）MTU:1500（最大传输单元）：1500字节

第四、五行：接收、发送数据包情况统计

第七行：接收、发送数据字节数统计信息







