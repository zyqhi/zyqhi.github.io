---
title: usbmux协议分析
tags: usbmuxd
---

# usbmux协议介绍

初闻usbmux协议时，可能会让人感觉比较陌生，但其实如果你是MAC用户的话，你可能每天都在和它打交道，只是不知道而已。当通过USB将iPhone连接到MAC时，usbmux协议已经在背后为你默默工作了。

usbmux是苹果的私有协议，苹果设计该协议的原因是为了自家的macOS APP能够和iDevice进行通信，从而实现诸如iTunes备份iPhone、Xcode真机调试等功能。只是后来该协议被开发者破解了，于是为众人所知。

usbmuxd是对usbmux协议在macOS平台的上实现，也是macOS系统上的一个守护进程（Daemon），它随着系统的启动而启动。在macOS上，与usbmuxd相关的文件主要有3个：

1. /System/Library/PrivateFrameworks/MobileDevice.framework/Versions/A/Resources/usbmuxd
2. /System/Library/LaunchDaemons/com.apple.usbmuxd.plist
3. /var/run/usbmuxd

其中**/System/Library/PrivateFrameworks/MobileDevice.framework/Versions/A/Resources/usbmuxd**为可执行文件。 

**/System/Library/LaunchDaemons/com.apple.usbmuxd.plist**为usbmuxd服务的开机启动配置文件，如图所示。launchd在系统启动时会根据该文件的配置信息，启动usbmuxd服务，创建对应的socket（也即/var/run/usbmuxd）。

![image-20190820203643037](http://ww4.sinaimg.cn/large/006y8mN6ly1g67kytqiw2j31ff0u07g0.jpg)

**/var/run/usbmuxd**在开机启动时创建，该文件其实是一个Unix domain socket，用于usbmuxd进程和其他进程——比如Xcode、iTunes等——之间的进程间通信（IPC）。

``` shell
➜  zyqhi.github.io git:(master) ✗ file /var/run/usbmuxd
/var/run/usbmuxd: socket
```



# 基于usbmux协议的设备间通信

前文也提到，苹果设计usbmux协议是为了解决macOS APP和iOS设备之间的通信问题。如果只是普通的协议的话，倒也没什么可说的，只是该协议呢，提供了一种类似TCP socket的API，使得macOS和iOS设备之间的通信，如同是网络上的两个主机之间的通信。

> During normal operations, [iTunes](https://www.theiphonewiki.com/wiki/ITunes) communicates with the [iPhone](https://www.theiphonewiki.com/wiki/List_of_iPhones) using something called “usbmux” – this is a system for multiplexing several “connections” over one USB pipe. Conceptually, it provides a TCP-like system – processes on the host machine open up connections to specific, numbered ports on the mobile device. (This resemblance is more than superficial – on the mobile device, usbmuxd actually makes TCP connections to localhost using the port number you give it.)
>
> FROM: https://www.theiphonewiki.com/wiki/Usbmux



说完usbmux协议的用途，我们再来看看usbmux协议是怎样工作的。我们以iTunes和iOS通信为例，来看下整个通信架构，如图所示：

![image-20190821135515906](http://ww2.sinaimg.cn/large/006y8mN6ly1g67l0urzuyj30z90hit9x.jpg)

从图中可以看出，整个通信架构和经典的C/S架构非常类似。其中iOS端的服务（lockdown）的角色是服务端，macOS端的程序（iTunes）则为客户端。

## lockdown服务

先来看下iOS端：

图中的lockdown是iOS端的系统服务，它在iOS系统启动时，由launchd进程启动。lockdown服务对应的启动配置文件在iOS系统上的完整路径为：**/System/Library/LaunchDaemons/com.apple.mobile.lockdown.plist**。

![image-20190821135906471](http://ww2.sinaimg.cn/large/006y8mN6ly1g67l12s8wij31eo0t2qev.jpg)

lockdown作为一个网关，用于协调macOS APP和iOS其他服务之间的通信。举个例子，Xcode若要执行真机调试，首先需要和lockdown服务通信，发出启动调试请求，lockdown收到请求以后，启动iOS端对应的调试服务（debugserver），然后Xcode便与debugserver之间建立了通信连接。

lockdown服务启动以后，会创建一个TCP listen socket，端口号为62078，地址为本机地址：127.0.0.1。这一点我们可以在越狱机器上得到验证：

![image-20190821141233973](http://ww3.sinaimg.cn/large/006y8mN6ly1g67l16se48j31ej0u0qhl.jpg)

对于iOS端的服务而言，和普通的TCP Server没有差别。

> 关于lockdown服务的更多内容，可以参考这篇文章：[Understanding usbmux and the iOS lockdown service](https://medium.com/@jon.gabilondo.angulo_7635/understanding-usbmux-and-the-ios-lockdown-service-7f2a1dfd07ae)



## usbmuxd服务

再来看macOS端：

前文讲过，usbmuxd是macOS的系统服务，该服务为macOS端的通信代理，当其他应用——比如iTunes——需要与iOS设备通信时，只需要与usbmuxd服务通信即可，usbmuxd负责将iTunes的通信请求转发到iOS端。usbmuxd服务与iOS设备通信基于USB通信协议。

iTunes与usbmuxd服务之间通过[Unix Domain Socket](https://en.wikipedia.org/wiki/Unix_domain_socket)进行通信，Unix Domain Socket的API和TCP Socket非常类似。来看下与usbmuxd协议之间建立连接的代码：

``` objc
int connect_to_usbmuxd() {
    // Create Unix domain socket
    int fd = socket(AF_UNIX, SOCK_STREAM, 0);
    // prevent SIGPIPE
    int on = 1;
    setsockopt(fd, SOL_SOCKET, SO_NOSIGPIPE, &on, sizeof(on));

    // Connect socket
    struct sockaddr_un addr;
    addr.sun_family = AF_UNIX;
    // 这个路径就相当于ip地址，为了理解这个过程，可以和TCP进行对比
    strcpy(addr.sun_path, "/var/run/usbmuxd");
    socklen_t socklen = sizeof(addr);

    if (connect(fd, (struct sockaddr *)&addr, socklen) == -1) {
        printf("Connect failure, fd is: %d.\n", fd);
    } else {
        printf("Connect successifully, fd is: %d.\n", fd);
    }

    return fd;
}
```

和TCP的差别有以下几点：

1.  domian参数为`AF_UNIX`，而非熟悉的`AF_NET`；
2. 通信地址不是IP地址，而是路径`/var/run/usbmuxd`；
3. 不需要指定端口。

初次之外，整个API的使用方式则几乎一样。iTunes在建立和usbmuxd之间的连接时，应该使用了类似的代码。



## 从一端到另一端

前文中提到，对于macOS端的程序而言，如果要与iOS上的程序通信，只需要和usbmuxd服务通信即可。而在上一节我们介绍了应用程序和usbmuxd服务建立连接的方式，那么假设iTunes通过上述方式建立了与usbmuxd之间的连接，那么接下来iTunes是不是就可以直接与位于iOS端的lockdown服务通信了呢？

答案是NO。

在回答为什么是NO之前，先来回忆一下基于socket的TCP通信过程。我们知道在TCP/IP通信协议体系里，IP抽象了网络上两台主机之间的通信，而TCP抽象了两台主机上应用程序之间的通信。IP地址表示了网络中的一台主机，而端口号则表示应用程序。

而在上一节，在建立应该程序与代理（usbmuxd）之间的连接时，并未指定端口号。未指定端口号，那么usbmuxd服务便不知道应用程序要和iOS端的那个应用程序进行通信。因此我们此时还无法和iOS端的应用程序通信。比如lockdown服务的端口号是62078，而iTunes在建立连接时并未指定该端口号，因此usbmuxd根本不知道iTunes要和lockdown服务通信。

其实，如果用上一节中的提到的建立连接，只是建立了macOS应用程序与代理的通信连接，该连接建立以后，应用程序能够和代理进行通信，但如若要和iOS上的应用程序通信，我们还需要对代理进行配置，**让代理和iOS端的应用程序建立连接**，整个通信链路才算打通。

下面我们便来看下如何『配置』代理，让其和iOS端应用程序建立连接。

### Listen

在将具体的连接过程之前，我们先来想下，如果要usbmuxd服务和iOS端应用程序之间建立连接，必须要满足哪些条件：

1. 一台Mac可以连接多台iOS设备，连接时需要知道和哪一台iOS设备建立连接，因此需要device id来唯一标识一台设备；
2. 建立连接时必须要知道当前Mac是否连接到iOS设备，也即需要监听iOS设备的插拔；
3. usbmuxd不会主动去连接iOS端上的应用程序，必须macOS应用程序驱使其完成，也即对其进行配置，此时macOS应用程序和usbmuxd之间需要一套通信协议；
4. 必须指定需要连接到的服务的端口，这样才能唯一标识一个服务。

先看问题3，Apple定义了一套配置协议，该协议的数据传输格式为plist，并被破解，协议的具体细节会在下文中阐释数据收发过程中讲到。而对于4，后文中会给出解答，现在先忽略。

而对于问题1和2，usbmuxd已经具备该功能，但是如果要『看到』该功能，我们需要向usbmuxd发送命令，这便是本节中要讲的**Listen**命令。

所谓发送Listen命令，便是向上一节中`connect_to_usbmuxd`函数创建的socket文件描述符中写入一段数据（控制命令），然后读一段数据（usbmuxd的返回值），读取的数据。发送逻辑的代码如下：

```objc
void send_listen_packet() {
    listen_channel = connect_to_usbmuxd_channel(); // 该函数调用 connect_to_usbmuxd 创建socket，并根据socket文件描述符创建一个dispatch I/O对象
    
    // 1. Listen指令对应的键值对
    NSDictionary *packet = @{
                             @"ClientVersionString": @"1",
                             @"MessageType": @"Listen",
                             @"ProgName": @"Peertalk Example"
                             };
    NSLog(@"send listen packet: %@", packet);
    send_packet(packet, 0, listen_channel);
}

void send_packet(NSDictionary *packetDict, int tag, dispatch_io_t channel) {
    NSData *plistData = [NSPropertyListSerialization dataWithPropertyList:packetDict format:NSPropertyListXMLFormat_v1_0 options:0 error:NULL];
    
    int protocol = USBMuxPacketProtocolPlist;
    int type = USBMuxPacketTypePlistPayload;
    
    // 2. 封装成usbmux协议要求的格式
    usbmux_packet_t *upacket = usbmux_packet_create(
                                                    protocol,
                                                    type,
                                                    tag,
                                                    plistData ? plistData.bytes : nil,
                                                    (uint32_t)(plistData.length)
                                                    );
    
    dispatch_data_t data = dispatch_data_create((const void*)upacket, upacket->size, usbmuxd_io_queue, ^{
        usbmux_packet_free(upacket);
    });
    
    dispatch_io_write(channel, 0, data, usbmuxd_io_queue, ^(bool done, dispatch_data_t data, int _errno) {
        NSLog(@"dispatch_io_write: done=%d data=%p error=%d", done, data, _errno);
        if (!done) { return; }
    });
}
```

这部分代码核心就2处，代码中已标出：

1. 以plist格式封装Listen指令；
2. 是将封装后的Listen报文发送到usbmuxd。

对应的报文格式如下图所示：

![image-20190821164908991](http://ww4.sinaimg.cn/large/006y8mN6ly1g67l1gf4fij319m0kxgnn.jpg)

Listen命令发送成功以后，每当有iOS设备插拔，usbmuxd便会向我们之前通过`connect_to_usbmuxd`函数创建的socket文件描述符中写入数据。读取逻辑如下：

```objc
void read_packet_on_channle(dispatch_io_t channel) {
    // 1. Read the header
    usbmux_packet_t ref_upacket;
    dispatch_io_read(channel, 0, sizeof(ref_upacket.size), usbmuxd_io_queue, ^(bool done, dispatch_data_t  _Nullable data, int error) {
        
        if (!done) { return; }
        
        // Read size of incoming usbmux_packet_t
        uint32_t upacket_len = 0;
        char *buffer = NULL;
        size_t buffer_size = 0;
        // data 是读取到的数据，这一步获取到读取到的data的长度，并将buffer指向对应的缓冲区
        dispatch_data_t map_data = dispatch_data_create_map(data, (const void **)&buffer, &buffer_size);
        memcpy((void *)&(upacket_len), (const void *)buffer, buffer_size);
        
        // Allocate a new usbmux_packet_t for the expected size
        uint32_t payloadLength = upacket_len - (uint32_t)sizeof(usbmux_packet_t);
        usbmux_packet_t *upacket = usbmux_packet_alloc(payloadLength);
        
        // 2. Read rest of the incoming usbmux_packet_t
        off_t offset = sizeof(ref_upacket.size);
        dispatch_io_read(channel, offset, upacket->size - offset, usbmuxd_io_queue, ^(bool done, dispatch_data_t data, int error) {
            NSLog(@"dispatch_io_read %lld,%lld: done=%d data=%p error=%d", offset, upacket->size - offset, done, data, error);
            
            if (!done) { return; }
            
            // Copy read bytes onto our usbmux_packet_t
            char *buffer = NULL;
            size_t buffer_size = 0;
            dispatch_data_t map_data = dispatch_data_create_map(data, (const void **)&buffer, &buffer_size);
            memcpy(((void *)(upacket))+offset, (const void *)buffer, buffer_size);
            NSLog(@"package protocol is: %u, type is: %u", upacket->protocol, upacket->type);
            
            // 3. Try to decode any payload as plist
            NSError *err = nil;
            NSDictionary *dict = nil;
            if (usbmux_packet_payload_size(upacket)) {
                dict = [NSPropertyListSerialization propertyListWithData:[NSData dataWithBytesNoCopy:usbmux_packet_payload(upacket) length:usbmux_packet_payload_size(upacket) freeWhenDone:NO] options:NSPropertyListImmutable format:NULL error:&err];
            }
            
            // 4. Read next
            read_packet_on_channle(channel);
            
            usbmux_packet_free(upacket);
        });
    });
}
```

读逻辑中我删除了校验和容错逻辑，只保留了比较关键的部分，Listen命令的读操作说明如下：

1. 先读取报文头，usbmuxd返回的报文格式也遵循usbmux协议，**头部中记录了报文载荷的长度**，所以必须先读出报文头之后，才能知道载荷的边界到哪里；
2. 从头部解析出载荷长度之后，读取报文载荷数据；
3. 头部和载荷读取完成之后，构成一个完整的载荷，此时以plist格式解析载荷，获取返回内容；
4. 读取下一个报文，读操作采用的I/O模式是阻塞模式，一直监听是否有数据，当有数据时便进行读取解析。

![image-20190821171830735](http://ww1.sinaimg.cn/large/006y8mN6ly1g67l1khc1cj319m0rrad3.jpg)

如图所示，是当iOS设备通过USB连接到（Attached）Mac时，usbmuxd服务返回的数据。其中比较关键的字段有两个：

1. `MessageType`: Attached表明有设备连接的Mac，而Detached则表明断开连接；
2. `DeviceID`: 标识当前插入的设备，后续与设备建立连接时会用到。

iTunes监听设备插拔时，便是通过类似的方式。

### Connect

说完Listen，再来看下Connect。在上一节中，我们抛出了usbmuxd服务和iOS端应用程序之间建立连接的4点要求，其中前3个在上一节都已经得到解决，而第4个问题——端口号——其实在lockdown的启动配置文件时就已经知道了。此时我们已经具备建立连接的所有条件：

1. Listen时得到DeviceID；
2. lockdown启动配置文件中得知lockdown的端口号。

在讲Listen时，我们知道可以通过想usbmuxd服务发送Listen命令监听iOS设备插拔，类似地，也可以通过向usbmuxd服务发送Connect命令对其配置，使其建立和iOS端应用程序的连接。

整个过程如下，我们需要调用`connect_to_usbmuxd`函数再创建一个socket，用作Connect命令，和Listen命令的发送方式一样，发送Connect命令也是向socket文件描述符中写入数据：

```objc
void send_connect_usb_packet() {
    print_empty_lines();
    
    connect_channel = connect_to_usbmuxd_channel();
    

    port = ((port<<8) & 0xFF00) | (port>>8);
    NSDictionary *packet = @{
                             @"ClientVersionString" : @"1",
                             @"DeviceID" : deviceID,
                             @"MessageType" : @"Connect",
                             @"PortNumber" : [NSNumber numberWithInt:port],
                             @"ProgName" : @"Peertalk Example"
                             };
    
    NSLog(@"send connect to usb packet: %@", packet);
    send_packet(packet, 1, connect_channel);
    read_packet_on_channle(connect_channel);
}
```

其中关键的字段有：

1. `DeviceID`: 设备id，在Linten时获得；
2. `MessageType`: 消息类型，Connect表示需要建立连接；
3. `PortNumber`: 端口号，对应iOS端目标连接服务的端口，以lockdown为例，为62078。

对应的报文格式为：

![image-20190821193559069](http://ww4.sinaimg.cn/large/006y8mN6ly1g67l1p6dy2j319m0kxmyn.jpg)

> 注：实际发送过程中此处的PortNumber会以网络字节序的形式传输。

发送Connect命令之后，如果连接成功，此时便会收到usbmuxd返回的数据（读取Connect返回的数据和之前的读取Listen命令返回的数据方式相同）：

![image-20190821193827916](http://ww2.sinaimg.cn/large/006y8mN6ly1g67l1sgctij319m0fo3z8.jpg)

Number为0表示连接成功。

为了容易理解，我们来和标准的TCP通信做个对比，其中右侧的lockdown，就是标准的TCP Server，而左侧的iTunes，建立与lockdown的连接时不是像标准的TCP Client一样，调用connect函数，而是向usbmuxd写入一段数据，但是，虽然方法不同，但是二者在通信意义上是等价的。

![image-20190821204733214](http://ww3.sinaimg.cn/large/006y8mN6ly1g67l7tqac1j30x70t0tdb.jpg)

并且，当在Connect命令发送后，在iOS侧，lockdown的accept方法会执行，随后双方便可以像标准的TCP通信一样发送和接收数据。

iTunes发送数据的方式如下：

``` objc
void send_msg(NSString *msg) {
    dispatch_data_t payload = create_payload(msg);
    dispatch_data_t frame = create_frame(101, 0, payload);

    // send through connect channel, not tcp_channel
    dispatch_io_write(connect_channel, 0, frame, usbmuxd_io_queue, ^(bool done, dispatch_data_t  _Nullable data, int error) {
        NSLog(@"error is: %d", error);
    });
}
```

其中connect_channel是我们在发送Connect命令时创建的文件描述符。

注意Listen和Connect使用不同的文件描述符。
{:.warning}

经历以上过程，macOS程序便实现了通过usbmuxd和iOS通信。总结下来：

1. macOS端应用通过Unix Domain Socket以类似TCP的方式和usbmuxd服务进行通信；
2. macOS端应用程序通过向usbmuxd服务发送Connect命令，建立与iOS端目标进程之间的通信连接，端到端的通信基于端口号；
3. iOS端程序以标准的TCP Server形式存在；
4. usbmuxd作为中间代理，封装了USB协议的通信细节，对上提供类TCP的接口；
5. 基于usbmux协议和usbmuxd服务，实现了macOS程序和iOS端程序之间简单、高效的通信。

> 简单在于协议的API基于socket，为大众熟知，而在通信效率方面，基于USB协议的通信效率显然比基于TCP/IP网络要快得多。

# usbmux应用

如果是单纯分析macOS程序和iOS程序之间的通信原理的话，那大可不必如此大费周章，usbmuxd的魔力不仅在于它可以用于苹果自带应用和iOS程序之间的通信，而且，我是说而且，它可以用于为我们所用，基于usbmuxd服务，我们可以利用usbmux协议构建我们自己的应用。

{:.success}

我们可以基于usbmuxd服务，利用usbmux协议构建我们自己的应用。

## 重新审视lockdown

我们回过头来，重新看下lockdown做了哪些事情，从前文中lockdown的启动配置文件可以看出，它在开机时创建了一个listen socket，监听本地端口：62078，然后像普通的TCP Server那样，bind、accept阻塞等待客户端的连接。而当macOS端向usbmuxd发送Connect命令时，lockdown的accept结束阻塞，连接建立。

> lookdown的源码没有，但是从启动配置文件和nc端口测试，结合网络知识，我们基本可以确认lockdown的内在逻辑。

那么，如果我们模拟lockdown，新建一个iOS APP，指定端口号，比如2345，然后通过调用socket、bind、listen、accept建立我们自己的TCP Server监听127.0.0.1:2345。

同时新建一个macOS应用，像iTunes一样连接usbmuxd服务，以端口号2345向其发送Connect命令，那么会发生什么呢？

答案是，如我们所愿，利用usbmuxd协议，我们建立了自己的macOS应用和iOS APP之间的通信连接，随后便可在此协议的基础之上，实现自建iOS APP和macOS应用程序之间的高效通信。整个通信框架如下：

![image-20190821201547383](http://ww3.sinaimg.cn/large/006y8mN6ly1g67l20gsdkj30z90hiaba.jpg)



## usbmux应用举例

usbmux协议由于通信效率较高，在数据量较大的通信场景中，体验非常突出，毕竟当传输的数据量较大时，效率就是体验。目前基于usbmux有非常多的有意思的应用，常见的有：

- [PeerTalk](https://github.com/rsms/peertalk): PeerTalk基于usbmux实现了一个简单的IM，可用于macOS端和和iOS端之间聊天。本文的代码就是参照PeerTalk，如果你想基于usbmux协议构建自己的应用，而又不想深入了解usbmux协议的的话，可以直接搬运PeerTalk的代码。
- [lookin]([https://lookin.work](https://lookin.work/)): 腾讯开发的iOS界面调试利器，简直比Reveal还好用，而且免费。 

# 参考

- [usbmux_demo](https://github.com/zyqhi/usbmux_demo):  本文中的示例代码。
- [usbmux](https://www.theiphonewiki.com/wiki/Usbmux)
- [launchd](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
- [Understanding usbmux and the iOS lockdown service](https://medium.com/@jon.gabilondo.angulo_7635/understanding-usbmux-and-the-ios-lockdown-service-7f2a1dfd07ae)

