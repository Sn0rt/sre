# 0x00 Introduction

## 0x01 what is netlink?

netlink 是一种用户进程和内核,或者进程之间的沟通机制,不能用于跨主机的沟通.

## 0x02 the advantage of netlink?

多数的 Linux 内核态程序都需要和用户空间的进程交换数据,但是传统的 Unix 的 IPC (各类管道、消息队列、内存共享、信号量)机制不能为进程与内核通讯提供足够的支持.
不过 Linux 提供了很多与内核沟通的方法如内核启动参数、模块参数与 sysfs、sysctl、系统调用、netlink、procfs、seq_file、debugfs 和 relayfs.

|沟通方法          | 应用场景
--------------------|--------
|内核启动参数       | 内核开发者可以通过这种方式来向内核传输数据，从而控制内核启动行为.
|模块参数,sysfs    | 当内核部分子系统编译成模块,则可以通过命令行在插入模块时传递参数,或在运行时,通过sysfs来设置或读取模块数据.
|sysctl           | 其被应用来设置与获取运行时内核的配置参数,通过这种方式,用户应用可以在内核运行的任何时刻来改变内核的配置参数,也可以在任何时候获得内核的配置参数.
|系统调用          | 是内核提供给应用程序的接口,应用对底层硬件的操作大部分都是通过调用系统调用来完成的.
|netlink          | 是一种在内核与用户应用间进行双向数据传输的非常好的方式.
|procfs           | 是较老的用户态与内核态的数据交换方式,内核很多数据通过这种方式提供给用户,内核很多参数也是通过这种方式来用户方便设置,但其有一个缺陷,若输出内容大于1个内存页,需要多次读.
|debugfs          | 只有在需要的时候使用,它在需要时通过在一个虚拟文件系统中创建一个或多个文件来向用户空间应用提供调试信息.
|relayfs          | 是一个快速的转发（relay）数据的文件系统，它以其功能而得名。它为那些需要从内核空间转发大量数据到用户空间的工具和应用提供了快速有效的转发机制。

Netlink 相对于系统调用，ioctl 以及 /proc 文件系统而言具有以下优点：

* 为了使用 netlink,用户仅需要在 include/linux/netlink.h 中增加一个新类型的 netlink 协议定义即可, 如 #define NETLINK_MYTEST 17 然后,内核和用户态应用就可以立即通过 socket API 使用该 netlink 协议类型进行数据交换.但系统调用需要增加新的系统调用,ioctl 则需要增加设备或文件,  那需要不少代码,proc 文件系统则需要在 /proc 下添加新的文件或目录,那将使本来就混乱的 /proc 更加混乱.
* netlink是一种异步通信机制,在内核与用户态应用之间传递的消息保存在socket缓存队列中,发送消息只是把消息保存在接收者的socket的接收队列,而不需要等待接收者收到消息,但系统调用与 ioctl 则是同步通信机制,如果传递的数据太长,将影响调度粒度.
* 使用 netlink 的内核部分可以采用模块的方式实现,使用 netlink 的应用部分和内核部分没有编译时依赖,但系统调用就有依赖,而且新的系统调用的实现必须静态地连接到内核中,它无法在模块中实现,使用新系统调用的应用在编译时需要依赖内核.
* netlink 支持多播,内核模块或应用可以把消息多播给一个netlink组,属于该 neilink 组的任何内核模块或应用都能接收到该消息,内核事件向用户态的通知机制就使用了这一特性,任何对内核事件感兴趣的应用都能收到该子系统发送的内核事件,在后面的文章中将介绍这一机制的使用.
* 内核可以使用 netlink 首先发起会话,但系统调用和 ioctl 只能由用户应用发起调用.
* netlink 使用标准的 socket API,因此很容易使用,但系统调用和 ioctl则需要专门的培训才能使用.

## 0x03 netlink feature

netlink 只是框架提供基本的和内核沟通的功能,具体要做的事情由基于 netlink 的子协议做.
而内核中已经存在基于 netlink 的具体协议有`Linux/include/uapi/linux/netlink.h`:

    #define NETLINK_ROUTE		0	/* Routing/device hook				*/
    #define NETLINK_UNUSED		1	/* Unused number				*/
    #define NETLINK_USERSOCK	2	/* Reserved for user mode socket protocols 	*/
    #define NETLINK_FIREWALL	3	/* Unused number, formerly ip_queue		*/
    #define NETLINK_SOCK_DIAG	4	/* socket monitoring				*/
    #define NETLINK_NFLOG		5	/* netfilter/iptables ULOG */
    #define NETLINK_XFRM		6	/* ipsec */
    #define NETLINK_SELINUX		7	/* SELinux event notifications */
    #define NETLINK_ISCSI		8	/* Open-iSCSI */
    #define NETLINK_AUDIT		9	/* auditing */
    #define NETLINK_FIB_LOOKUP	10	
    #define NETLINK_CONNECTOR	11
    #define NETLINK_NETFILTER	12	/* netfilter subsystem */
    #define NETLINK_IP6_FW		13
    #define NETLINK_DNRTMSG		14	/* DECnet routing messages */
    #define NETLINK_KOBJECT_UEVENT	15	/* Kernel messages to userspace */
    #define NETLINK_GENERIC		16
    /* leave room for NETLINK_DM (DM Events) */
    #define NETLINK_SCSITRANSPORT	18	/* SCSI Transports */
    #define NETLINK_ECRYPTFS	19
    #define NETLINK_RDMA		20
    #define NETLINK_CRYPTO		21	/* Crypto layer */
    #define NETLINK_INET_DIAG	NETLINK_SOCK_DIAG

## 0x04 the architecture of netlink

<pre><code>
     +---------------------+      +---------------------+
      | (3) application "A" |      | (3) application "B" |
      +------+--------------+      +--------------+------+
             |                                    |
             \                                    /
              \                                  /
               |                                |
       +-------+--------------------------------+-------+
       |        :                               :       |   user-space
  =====+        :   (5)  Kernel socket API      :       +================
       |        :                               :       |   kernel-space
       +--------+-------------------------------+-------+
                |                               |
          +-----+-------------------------------+----+
          |        (1)  Netlink subsystem            |
          +---------------------+--------------------+
                                |
          +---------------------+--------------------+
          |       (2) Generic Netlink bus            |
          +--+--------------------------+-------+----+
             |                          |       |
     +-------+---------+                |       |
     |  (4) Controller |               /         \
     +-----------------+              /           \
                                      |           |
                   +------------------+--+     +--+------------------+
                   | (3) kernel user "X" |     | (3) kernel user "Y" |
                   +---------------------+     +---------------------+
</code></pre>

# 0x10 date struct

## 0x11 netlink protocol

在 linux 内核中每个协议都需要注册一个`net_proto_family`实例,该函数包含一个函数指针,在创建属于该协议族的 `socket` 的时候被调用, netlink 的这个函数指针是 `netlink_create`,该函数分配一个`struct sock`的实例,通过` socket->sk` 关联到` socket`, 不过这个函数不仅为`struct sock`分配了空间,也为`netlink_sock`分配了空间.

    struct netlink_sock {
	    /* struct sock has to be the first member of netlink_sock */
	    struct sock		sk;
	    u32			portid;
	    u32			dst_portid;
	    u32			dst_group;
	    u32			flags;
	    u32			subscriptions;
        ...
	    wait_queue_head_t	wait;
	    struct netlink_callback	*cb;
        ...
	    void			(*netlink_rcv)(struct sk_buff *skb);
	    void			(*netlink_bind)(int group);
        ...
    };

省略了一部分代码,保留了主题.
可以看见`sock`实例直接嵌入`netlink_sock`中,给出的一个`netlink`套接字的`struct sock`实例,与之相关联,特定于`netlink`的`netlink_socket`实例,可以使用`nlk_sk`获得.链接两端的端口ID分别保存在`portid`和`dst_portid`中.`netlink_rcv`是在`socket`接受到数据时候调用.

## 0x12 the address of socket

类似于其余网络协议,每个`netlink`套接字也需要分配一个地址,下列`struct sockaddr`的变体表示`netlink`地址:

    struct sockaddr_nl {
        __kernel_sa_family_t	nl_family;	/* AF_NETLINK	*/
	    unsigned short	nl_pad;		/* zero		*/
	    __u32		nl_pid;		/* port ID	*/
        __u32		nl_groups;	/* multicast groups mask */
    };

为区分具体的子协议内核使用了nl_family字段,<netlink.h>指定了不同的几种族,就是上面具体作用部分列出来的. nl_pad 是对其补全总是0. nl_pid为发送消息的进程pid,若是希望内核处理或者多播消息则置0,否则为处理消息的线程组id. 字段 nl_groups 用于指定播组, bind 函数用于把调用进程加入到该字段指定的播组,若是为0表示不加入任何播组.
若是一个进程的多个线程使用 netlink socket 的情况,进程的字段的nl_pid可以设置为其他值.因此字段`nl_pid`实际上未必是进程 ID,它只是用于区分不同的接收者或发送者的一个标识,用户可以根据自己需要设置该字段.

## 0x13 message format

<pre><code>
Message Format:
<--- nlmsg_total_size(payload)  --->
<-- nlmsg_msg_size(payload) ->
+----------+- - -+-------------+----+----------
| nlmsghdr | Pad |   Payload   |Pad | nlmsghdr
+----------+- - -+-------------+----+----------
nlmsg_data(nlh)---^                 ^
nlmsg_next(nlh)-----------------------+
</code></pre>
一个基本的消息单元有两个部分组成:头部与payload,且这个message对齐到`NLMSG_ALIGNTO`,一般是`#define NLMSG_ALIGNTO	4U`.

## 0x14 netlink messages header 

不同于BSD套接字，头信息中的标识和目的地都是自动生成, Netlink消息头（结构体nlmsghdr）必须由发送方准备好，就像socket工作在SOCK_RAW模式下 一样。尽管SOCK_DGRAM被用于创建它.

    struct nlmsghdr {
	    __u32		nlmsg_len;	/* Length of message including header */
	    __u16		nlmsg_type;	/* Message content */
        __u16		nlmsg_flags;	/* Additional flags */
	    __u32		nlmsg_seq;	/* Sequence number */
	    __u32		nlmsg_pid;	/* Sending process port ID */
    };
    
`nlmsg_type`是基于`netlink`的协议的私有的,`netlink`协议不去检查子协议.
`nlmsg_flags`的类型都定义在`netlink.h`里面了,一般情况下只要关注:如果消息包含一个请求,要求执行特定的操作(而不是传输一些状态信息),那么`NLM_F_REQUEST`将被置位,而`NLM_F_ACK`要求在接受上述消息并成功返回处理请求之后发送一个`ack`.

标准的flages,其余的并没有列出了来`linux/include/uapi/linux/netlink.h`.

    #define NLM_F_REQUEST		1	/* It is request message. 	*/
    #define NLM_F_MULTI		2	/* Multipart message, terminated by NLMSG_DONE */
    #define NLM_F_ACK		4	/* Reply with ack, with zero or error code */
    #define NLM_F_ECHO		8	/* Echo this request 		*/
    #define NLM_F_DUMP_INTR		16	/* Dump was inconsistent due to sequence change */

## 0x15 netlink messages payload

按照rfc里面的定义的服务模型,payload部分就是基于netlink的子协议的数据,各个具体的子协议不同.

# 0x20 How to use netlink?

## 0x21 in userspace

在用户态应用使用标准的socket与内核通讯,标准的 socket API 的函数, socket(), bind(), sendmsg(), recvmsg() 和 close()很容易地应用到 netlink socket。
为了创建一个 netlink socket，用户需要使用如下参数调用 socket():

    socket(AF_NETLINK, SOCK_RAW, netlink_type)
    
用户态更多的应用场景是使用`libnl`库[这里](https://www.infradead.org/~tgr/libnl/).
    
## 0x22 in kernelspace

netlink的内核实现在`net/netlink/af_netlink.c`中,内核模块要想使用netlink,也必须包含头文件`linux/netlink.h`.内核使用netlink需要专门的API,这完全不同于用户态应用对netlink的使用.
如果用户需要增加新的netlink协议类型,必须通过修改`linux/netlink.h`来实现,当然,目前的netlink实现已经包含了一个通用的协议类型NETLINK_GENERIC以方便用户使用,用户可以直接使用它而不必增加新的协议类型.


### reference
[1] [http://linux.die.net/man/7/netlink](http://linux.die.net/man/7/netlink) 
[2] [Linux 系统内核空间与用户空间通信的实现与分析](https://www.ibm.com/developerworks/cn/linux/l-netlink/ )
[3] [在 Linux 下用户空间与内核空间数据交换的方式](https://www.ibm.com/developerworks/cn/linux/l-kerns-usrs/)
[4] [rfc3549](http://www.faqs.org/rfcs/rfc3549.html)
[5] [linuxjournal](http://www.linuxjournal.com/article/7356)
[6] [LWN](http://lwn.net/Articles/208755/)
