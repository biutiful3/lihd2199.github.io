---
title: "tcp"
date: 2021-08-30T20:27:54+08:00
draft: false
audthor: lihongda
categories: ["计算机网络"]
tags: ["运输层","tcp"]
---

### 一、tcp概述

1、**TCP是面向连接的运输层协议**

2、每一条***TCP连接只能有两个端点***(endpoint)，每一条TCP连接只能是点对点的（一对一）

3、**TCP提供可靠交付的服务，通过TCP连接传送的数据，无差错、不丢失、不重复、并且按序到达**

4、TCP提供**全双工通信**，TCP允许通信双方的应用进程在任何时候都能发送数据

5、**<u>面向字节流，TCP中的流(stream)指的是流入到进程或从进程流出的字节序列。“面向字节流”的含义是：虽然应用程序和TCP的交互是一次一个数据块，但TCP把应用程序交下来的数据看成仅仅是一连串的无结构的字节流。TCP并不知道所传送的字节流的含义。TCP不保证接收方应用程序所收到的数据块和发送方应用程序所发出的数据块具有对应大小的关系（例如，发送方应用程序交给发送方的TCP共10个数据块，但接收方的TCP可能只用了4个数据块就把收到的字节流交付上层的应用程序）。但接收方应用程序收到的字节流必须和发送方应用程序发出的字节流完全一样。当然，接收方的应用程序必须有能力识别收到的字节流，把它还原成有意义的应用层数据。</u>**


![](/img/tcp面向字节流.png)



6、TCP连接的端点叫做套接字(socket)或插口，端口号拼接到P地址即构成了套接字，同一个IP地址可以有多个不同的TCP连接，而同一个端口号也可以出现在多个不同的TCP连接中。



### 二、TCP报文格式

TCP虽然是面向字节流的，但TCP传送的数据单元却是报文段，一个TCP报文段分为首部和数据两部分。

TCP报文段首部的前20个字节是固定的，后面有4n字节是根据需要而增加的选项。因此**TCP首部的最小长度是20字节**。

![](/img/tcp首部.png)

1、源端口和目的端口 各占2个字节，分别写入源端口号和目的端口号

2、**<u>序号占  4字节。序号范围是[0, 2^32 - 1]，共2^32个序号。序号增加到2^32 - 1后，下一个序号就又回到0。也就是说，序号使用mod 2^32运算。TCP是面向字节流的。在一个TCP连接中传送的字节流中的每一个字节都按顺序编号。整个要传送的字节流的起始序号必须在连接建立时设置。首部中的序号字段值则指的是本报文段所发送的数据的第一个字节的序号。例如，一报文段的序号字段值是301，而携带的数据共有100字节。这就表明：本报文段的数据的第一个字节的序号是301，最后一个字节的序号是400。显然，下一个报文段（如果还有的话）的数据序号应当从401开始，即下一个报文段的序号字段值应为401。这个字段的名称也叫做“报文段序号”</u>**

3、确认号 占4字节，(ack)是期望收到对方下一个报文段的第一个数据字节的序号。例如，B正确收到了A发送过来的一个报文段，其序号字段值是501，而数据长度是200字节（序号501～700），这表明B正确收到了A发送的到序号700为止的数据。因此，B期望收到A的下一个数据序号是701，于是B在发送给A的确认报文段中把确认号置为701。请注意，现在的确认号不是501，也不是700，而是701。

4、数据偏移 占4位，这个字段实际上是指出TCP报文段的首部长度。“数据偏移”的单位是32位字。由于4位二进制数能够表示的最大十进制数字是15，因此数据偏移的最大值是60字节，这也是TCP首部的最大长度（即选项长度不能超过40字节）

5、**<u>确认ACK  仅当ACK = 1时确认号字段才有效。TCP规定，在连接建立后所有传送的报文段都必须把ACK置1</u>**

6、**<u>同步SYN  在连接建立时用来同步序号。当SYN = 1而ACK= 0时，表明这是一个连接请求报文段。对方若同意建立连接，则应在响应的报文段中使SYN = 1和ACK = 1。因此，SYN置为1就表示这是一个连接请求或连接接受报文</u>**

7、**<u>终止FIN  用来释放一个连接。当FIN = 1时，表明此报文段的发送方的数据已发送完毕，并要求释放运输连接</u>**

8、窗口 占2字节。窗口值是[0, 2^16 - 1]之间的整数。窗口指的是 **<u>发送本报文段的一方的接收窗口</u>**（而不是自己的发送窗口）。**<u>窗口值告诉对方：从本报文段首部中的确认号算起，接收方目前允许对方发送的数据量</u>**。之所以要有这个限制，是因为接收方的数据缓存空间是有限的，总之，窗口值作为接收方让发送方设置其发送窗口的依据

### 三、TCP可靠传输

#### 滑动窗口

发送方A的发送窗口：在没有收到B的确认的情况下，A可以连续把窗口内的数据都发送出去。凡是已经发送过的数据，在未收到确认之前都必须暂时保留，以便在超时重传时使用

**<u>发送窗口里面的序号表示允许发送的序号。发送窗口后沿的后面部分表示已发送且已收到了确认。这些数据显然不需要再保留了。而发送窗口前沿的前面部分表示不允许发送的，因为接收方都没有为这部分数据保留临时存放的缓存空间。</u>**


![滑动窗口](/img/滑动窗口.jpg)


B的接收窗口：B的接收窗口大小是20，在接收窗口外面，到33号为止的数据是已经发送过确认，并且已经交付主机了。因此在B可以不再保留这些数据。接收窗口内的序号（34～53）是允许接收的。B收到了序号为37和38的数据，这些数据没有按序到达，B只能对按序收到的数据中的最高序号给出确认，因此B发送的确认报文段中的确认号仍然是37（即期望收到的序号），而不能是38或39。

**发送缓存用来暂时存放：**

(1) 发送应用程序传送给发送方TCP准备发送的数据；

(2) TCP已发送出但尚未收到确认的数据。

**接收缓存用来暂时存放：**

(1) 按序到达的、但尚未被接收应用程序读取的数据；

(2) 未按序到达的数据。

#### 超时重传

TCP的发送方在规定的时间内没有收到确认就要重传已发送的报文段

#### 流量控制

所谓流量控制(fow control)就是让发送方的发送速率不要太快，要让接收方来得及接收。

设A向B发送数据。在连接建立时，B告诉了A：“我的接收窗口rwnd = 400”（这里rwnd表示receiver window）。因此，发送方的发送窗口不能超过接收方给出的接收窗口的数值。请注意，**TCP的窗口单位是字节，不是报文段**。

TCP为每一个连接设有一个持续计时器，只要TCP连接的一方收到对方的零窗口通知，就启动持续计时器。若持续计时器设置的时间到期，就发送一个零窗口探测报文段（仅携带1字节的数据），而对方就在确认这个探测报文段时给出了现在的窗口值

#### 拥塞控制

所谓拥塞控制就是防止过多的数据注入到网络中，这样可以使网络中的路由器或链路不致过载。拥塞控制所要做的都有一个前提，就是网络能够承受现有的网络负荷。

### 四、连接与释放

#### 三次握手

1、A的TCP客户进程也是首先创建传输控制模块TCB，然后向B发出连接请求报文段，这时首部中的同步位SYN = 1，同时选择一个初始序号seq = x。TCP规定，SYN报文段（即SYN = 1的报文段）不能携带数据，但要消耗掉一个序号。这时，TCP客户进程进入SYN-SENT（同步已发送）状态。

2、B收到连接请求报文段后，如同意建立连接，则向A发送确认。在确认报文段中应把SYN位和ACK位都置1，确认号是ack = x + 1，同时也为自己选择一个初始序号seq = y。请注意，这个报文段也不能携带数据，但同样要消耗掉一个序号。这时TCP服务器进程进入SYN-RCVD（同步收到）状态。**<u>ack是确认号，在首部的地方有。</u>**

3、TCP客户进程收到B的确认后，还要向B给出确认。确认报文段的ACK置1，确认号ack = y + 1，而自己的序号seq = x + 1。TCP的标准规定，ACK报文段可以携带数据。但如果不携带数据则不消耗序号，在这种情况下，下一个数据报文段的序号仍是seq = x + 1。这时，TCP连接已经建立，A进入ESTABLISHED（已建立连接）状态。

![滑动窗口](/img/三次握手.png)

**为什么A还要发送一次确认呢？这主要是为了防止已失效的连接请求报文段突然又传送到了B，因而产生错误。**

#### 四次挥手

1、A的应用进程先向其TCP发出连接释放报文段，并停止再发送数据，主动关闭TCP连接。A把连接释放报文段首部的终止控制位FIN置1，其序号seq =u，它等于前面已传送过的数据的最后一个字节的序号加1。这时A进入FIN-WAIT-1（终止等待1）状态，等待B的确认。请注意，TCP规定，FIN报文段即使不携带数据，它也消耗掉一个序号。

2、B收到连接释放报文段后即发出确认，确认号是ack = u + 1，而这个报文段自己的序号是v，等于B前面已传送过的数据的最后一个字节的序号加1。然后B就进入CLOSE-WAIT（关闭等待）状态。TCP服务器进程这时应通知高层应用进程，因而从A到B这个方向的连接就释放了，这时的TCP连接处于半关闭(half-close)状态，即A已经没有数据要发送了，但B若发送数据，A仍要接收。也就是说，从B到A这个方向的连接并未关闭，这个状态可能会持续一些时间。

3、若B已经没有要向A发送的数据，其应用进程就通知TCP释放连接。这时B发出的连接释放报文段必须使FIN = 1。现假定B的序号为w（在半关闭状态B可能又发送了一些数据）。B还必须重复上次已发送过的确认号ack = u + 1。这时B就进入LAST-ACK（最后确认）状态，等待A的确认。

4、A在收到B的连接释放报文段后，必须对此发出确认。在确认报文段中把ACK置1，确认号ack = w + 1，而自己的序号是seq = u + 1（根据TCP标准，前面发送过的FIN报文段要消耗一个序号）。然后进入到TIME-WAIT（时间等待）状态。请注意，现在TCP连接还没有释放掉。必须经过时间等待计时器(TIME-WAIT timer)设置的时间2MSL后，A才进入到CLOSED状态。


![滑动窗口](/img/四次挥手.png)

