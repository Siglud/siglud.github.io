---
layout: post
title:  "Efficient IO with io_uring"
date:   2020-09-08 14:43:00 +0800
author: Siglud
categories:
  - Linux
tags:
  - Linux
comment: true
share: true
---

本文是 [https://kernel.dk/io_uring.pdf](https://kernel.dk/io_uring.pdf) 的翻译版本

本文旨在介绍最新的Linux  IO接口： io_uring，将其与其他现有的选择做比较。我们将会探究其存在的理由、内部工作原理以及用户可见的界面。本文不会涉及到其具体的命令，更多的时候关注于io_uring的工作原理，目的是希望读者对整个组件是如何工作的有一个深入的了解。本文和io_uring的man页面有一定的内容上的重复，这也是无法避免的。

## 1.0 引言

在Linux中，基于文件的IO有非常多种实现的方式，最基本的有 read 和 write 两个系统调用，还有增强的 pread 和 pwrite允许用户端传入 offset 参数，但是这依然显得不够用不够，于是后期增加了 preadv 和 pwritev 允许在一次函数调用中读、写多个非连续缓冲区（译者注：类似Java NIO里面的 Scatter/Gather IO）。除此之外，Linux还有 preadv2 和 pwritev2 系统调用，它们进一步的扩展了API以允许使用修饰符标识。虽然这些系统调用的有着各自不同的差异之处，但是他们都有着作为同步IO的共同特征——也就是系统只有在数据已经完全就绪的情况下才会给你返回值。在某些情况下这样并不是一个最好的选择，于是异步的接口诞生了，POSIX有着 aio_read 和 aio_write 去满足这样的需求。但是执行这些操作通常很麻烦，同时性能也很差。

Linux确实有一个原生的异步IO接口，名字就叫 aio，但是不幸的是，他受到了诸多限制：

* 最大的限制无疑是它仅支持 O_DIRECT （无缓存）访问。由于 O_DIRECT 本身的一些限制（比如它会跳过系统的Page Cache、对大小和对齐也有限制），这样就让它在很多应用场景下都显得很糟糕，而一旦切换为带缓存的IO，就又会变为同步方式运行。
*  就算是你满足了所有的限制条件让它得以异步的运行，但是你依然无法实现完全的异步。有非常多的情况下你还是不得不回到同步的IO——例如需要文件的元信息、尝试读取请求接口已满的存储设备（Linux对存储设备都有固定的读写数量限制）。这就意味着哪怕是你希望你请求的时候是完全异步的，但是你依然不得不在某些时候等待这些同步的任务做完。
* 这些API本身设计得不够好。每个IO提交需要复制64 + 8字节，每个完成的任务需要复制32个字节，于是一个声称支持零拷贝（Zero Copy）的IO却还是需要每次用掉104个字节。根据你的IO的大小，这个数字可能会是可观的（比如小文件大量读取、写入）。而事件循环缓冲区则更多时候让调用完成得更慢而且更难得正确。想要完成一次IO至少要使用两次系统调用（一次submit，一次wait-for-complete），在发现了CPU的Meltdown和Spectre漏洞之后，（这些安全补丁）更加严重的放缓了IO的速度。

多年以来，为了消除第一个限制作出了各种的努力，但是没有成功。而如今支持10ms以下延时的高IOPS的设备不断涌现（比如说M2接口的SSD），这才让老的IO接口显得更加不能跟上时代了。缓慢且不确定的提交延时成为了这类设备使用上的一个大问题。因为很难从单核上榨取更多的性能。所以这些aio应用被扔在了系统的犄角旮旯，伴随着一些可能长期无人发现但是随时可能出现的问题。

于是，一般的应用程序依然没有好的aio可用，Linux并没有提供给他们想要的功能。而事实上是不应该去由应用程序自己去创建和维护一些IO工具以获得高性能的IO的，特别是这些功能完全可以由内核提供的前提下。（应该是指的一些Kernel Bypass的黑科技）

## 2.0 改善现状

最初我们把目标定在提高现有的aio接口的性能方面，在我们彻底放弃之前，我们也确实走了很远。最初选择这个方向有一下几个原因：

* 如果可以扩展和改进现有的接口那铁定要比一个新的接口要好。采用新的接口要花时间，并且审核和批准一个新的接口是一项漫长而艰巨的任务。
* 一般而言，工作量会少很多。作为开发人员，大家总是希望能用更少的钱做更多的事情。扩展现有的接口无疑让你少写很多测试代码。

现有的 aio 接口是三个主要的系统调用组成的：一个用于设置 aio 上下文 io_setup(2)，一个用于IO请求提交io_submit(2)， 一个用于回调 io_getevents(2) 。如果一个改动需要变更这么多系统调用的话，我们必须要加入新的系统调用去传递参数，这样就不得不对同一段代码创建多个入口点、设置一些快捷的入口。结果代码会变得既复杂也难以维护。而且最终只能解决上一节所述的缺陷的其中之一。更重要的是还会让另一个缺陷变得更加糟糕——因为会让现有的API变得更难用更复杂。

所以最终的——虽然是一个艰难的决定，但是我们还是需要一些全新的东西来让我们实现所有的需要。我们需要它具有非常高的性能和可扩展性，同时也是容易使用的。并且弥补现有接口的缺陷。

## 3.0 新接口设计

从无到有做决定并非易事，这将允许我们享有充分的自由去提出新鲜的东西。根据重要性有高到低，主要的设计目标是：

* 易于使用，难以滥用。任何用户/应用程序的可见接口都应该以此为目标。接口应该易于理解和直观使用。
* 可扩展。虽然目标主要是存储相关，但是我希望它不仅仅能面向块IO，也就是说以后也可以支持网络或者非块IO，如果你做出了一个新的接口，这应该是理所当然（或者至少是尝试想要去）支持的。
* 功能丰富。当前的 aio 满足了一部分应用程序一部分的需要。我不想再创建一个功能仅能满足某些应用程序的需要，或者说需要一些应用程序去一遍又一便造轮子的接口（比如说IO线程池）
* 效率。尽管存储IO大部分还是基于块IO的，因此大小至少为512b或者4kb，操作这些大小的块的存储效率对某些应用程序而言至关重要。但是除此之外，某些请求甚至是没有携带数据块的，新的接口也必须高效的应对此类请求。
* 可伸缩性。尽管效率与低延迟很重要，特别是对于存储而言持续的提供最佳的性能也是同样重要的。我们一直在努力提供一种可伸缩的基础架构，使我们能够一直将这种可伸缩性提供给应用程序。

上面的某些目标似乎有些互斥，高效又可扩展的接口一般都挺难用的，更重要的是很难用得对。又丰富又高效的功能也很难实现，不过，这些依然是我们追求的目标。

## 4.0 进入 io_uring

尽管我们把易用性放在了首位，但是最初的设计还是围绕着效率进行的。毕竟效率这个东西绝对不可以容后再考虑必须从一开始就进行精心的设计。我清楚的知道我需要做到严格的Zero-Copy，在以前的 aio 的设计环节中就因为这个效率与可扩展性都明显受到了损害。

由于不希望做任何形式的内存拷贝，因此很明显，内核与应用程序必须要优雅地共享定义以下数据结构：IO本身与完成事件。如果你希望内核和应用系统共享那么多东西，那么很明显这些数据需要滞留在应用程序与内核的共享内存空间中。一旦这个也确定下来之后，那么很明显，你将需要协调应用程序与内核之间的同步。应用程序在不使用系统调用的情况下是无法和内核共享锁的，而系统调用肯定会减少与内核通讯的速度，这又与效率这一目标相悖。一种数据结构的出现满足了我们的需要：只有一个生产者和一个消费者的Ring Buffer（环形缓冲区，也称Circular Buffer）。通过共享 ring buffer我们就不用在应用程序与内核之间共享锁，巧妙的避免一些内存排序与内存屏障。

异步接口有两个基本的操作：提交请求与完成回调。对于提交请求而言，应用程序是生产者，而内核是消费者。反之则内核生产完成的信息而应用程序成为消费者。因此我们需要一对ring buffer 去作为这二者之间的高效联系渠道。这一对环就是这个新的接口的核心，io_uring。它们分别被命名为submission queue（SQ）和 completion queue（CQ）。

### 4.1 数据结构

我们确定了通讯的基础之后，就应该定义一下数据结构了，它应该着眼于完成一下目的：描述请求与完成事件。对于完成事件而言需要的数据是一目了然的。它需要携带与操作结果相关的信息，将请求结果的链接带给请求方。对于 io_uring，我们选择的结构如下：

```c
struct io_uring_cqe {
    __u64    user_data;
    __s32    res;
    __u32    flags;
};
```
io_uring这个名字我们现在应该都认识了，_cqe代表了Completion Queue Event（完成事件，下文简称cqe）。cqe包含了 *user_data* 这个字段，这个字段是来自与最初的提交的请求，并且可以包含应用程序识别所需的任何信息。一种常见的用法是使其成为原始请求的指针。内核不会去改动这个值，它会直接的从提交的请求处返回到完成事件的内容中。 *res* 中存放了请求的结果，可以把它想象成系统调用的返回值。对于一般的读写请求而言，它的返回值类似于 *read(2)* 或者 *write(2)*  的返回值。对于成功的请求而言，它会包含传输的字节总数。如果发生了错误，它会返回一个负数。例如，如果发生了I/O错误， *res* 将会是 *-EIO* 。最后， *flags* 将会包含一些操作相关的元信息。目前位置此字段尚未使用。

请求类型的定义更为复杂一些。它不仅需要描述比返回信息更多的信息。同时它还需要为 io_uring 未来能够扩展到一些别的类型的请求而做好准备。最终的结构如下：
```c
struct io_uring_sqe {
   __u8 opcode;
   __u8 flags;
   __u16 ioprio;
   __s32 fd;
   __u64 off;
   __u64 addr;
   __u32 len;
   union {
       __kernel_rwf_t rw_flags;
       __u32 fsync_flags;
       __u16 poll_events;
       __u32 sync_range_flags;
       __u32 msg_flags;   
   };
   __u64 user_data;
   union {
       __u16 buf_index;
       __u64 __pad2[3];
   };
};
```
与完成事件类似，提交侧的结构被成为Submission Queue Entry，或简称为 sqe。它包括一个 *opcode* 字段，描述事件的操作码（Operation code或简称op-code），比如 **IORING_OP_READV** ，标志着进行矢量读。 *flags* 字段包含一些应用修饰符，在之后的高级用例章节我们再详细解读它。 *ioprio* 是此次请求的优先级，对于一般的I/O读写而言，它遵循为 ioprio_set(2) 系统调用所用的定义。 *fd* 是请求关联的文件描述符， *off* 存放文件的偏移量，*addr* 包含了如果操作码描述了传输数据的操作，则该操作应在其中执行IO的地址。如果操作是某种形式的矢量读写操作，那么它将像 preadv(2) 一样指向一个类似 iovec array 结构体的指针。如果非矢量IO读写，那么 *addr* 必须是直接地址。这些设置将影响到len的定义， *len* 在非矢量IO读写时计算传输的字节数，而在矢量IO时记录 *addr* 所描述的矢量的数量。

接下来这个共用体主要包括了一些op-code的flag。例如，提示进行矢量读的（IORING_OP_READV），这些flag依然遵守preadv2（2）系统调用的约定。 *user_data* 就是上面提到的会在回调时传回的那个值，内核是不会去动这个值的，只是在回调的时候简单的拷贝过去。 *buf_index*  也将稍后在高级应用的章节详述。最后的共用体是一些填充物，这个主要是为了确保sqe的64位对齐（译者注：避免False Sharing），也为了能够给未来需要包含更多的请求数据留下一点空间——比如说，存放一些一些Key、Value存储的命令；或者说是一些应用程序事先对要写入的数据计算好的校验值来实现端对端的数据保护。

### 4.2 通道

上面说完了数据结构，让我们来介绍一下这两个环是怎么运作的。虽然提交和完成回调这两个环有点相似，但是这二者的索引还是有所不同的，我们先从简单的开始，完成回调环。

cqes 是一个数组，这段内存可以被应用程序和内核二者修改。当然了，如果内核正在写入cqes，那么只有内核可以实际修改cqes。这之间的交互是由环缓存来管理的。当一个新的事件被内核提交到CQ环它将修改数组尾部。而当应用程序消费它的时候，它会消费数组的头部。因此，如果头尾的地址不同，那么应用程序就知道还有足够的事件可以用来消费。这个环初始的时候有32位长，而当事件的长度超过了这个值的时候，环会自动变大，这个可以让我们充分的利用全部的环空间而不需要去管理”环已满“这样的错误提示，这样就降低管理环的复杂度，因此环的长度也必须是2的指数。

为了找到事件的索引，应用程序应该使用当前的尾部index去和整个ring的大小做与操作，一般的代码演示可是是下面这个样子的
```c
unsigned head;

head = cqring->head;
read_barrier();
if (head != cqring->tail) {
    struct io_uring_cqe *cqe;
    unsigned index;
    
    index = head & * (cqring->mask);
    cqe = &cqring->cqes[index];
    /* process completed cqe here */
    ...
    /* we've now consumed this entry */
    head++;
}

cqring->head = head;
write_barrier();
```

ring->cqes[] 是 io_uring_cqe 结构的共享数组。在下个部分中我们将深入讨论如何设置和管理这个共享内存（以及io_uring实例本身），以及它是如何魔法一般的实现读写的屏障的。

对于提交方而言，角色是相反的。应用程序负责更新尾部，而内核负责消费和更新头部。一个重要的不同是尽管CQ ring直接索引了cqes的共享数组，但是提交方还有一个间接的数组，提交侧的ring buffer存放这到这个数组的索引，该数组又包含了到sqes的索引。最初可能看起来很奇怪和让人困惑，但是有他们背后的一些道理。这允许开发者灵活的将请求单元嵌入内部数据结构中，同时保留了一次操作提交多个请求的能力。应用程序可以更加方便迁移到 io_uring 的接口。

增加一个供用户使用的sqe基本上是从内核中获取cqe的相反操作。一个典型的示例如下：
```c
struct io_uring_sqe *sqe;
unsigned tail, index;

tail = sqring->tail;
index = tail & (*sqring->ring_mask);
sqe = &sqring->sqes[index];

/* this call fills in the sqe entries for this IO */
init_io(sqe);

/* fill the sqe index into the SQ ring array */
sqring->array[index] = index;
tail++;

write_barrier();
sqring->tail = tail;
write_barrier();
```

与 CQ ring 相同，稍后将会说明读取和写入的屏障。上面是一个简化的例子，他假设当前 SQ ring 为空，或者至少它有空间还可以填充。

一旦内核消耗了 sqe，应用程序就可以自由的重用sqe条目——即使是内核并没有把这条sqe的条目完全做完。如果输入之后内核确实需要访问它，它就会制作一个稳定的副本。为什么会发生这种情况不一定很重要，但它对应用程序有一个重要的副作用。通常一个应用程序会要求一个给定大小的环，假设这个大小可能直接对应于应用程序在内核中可以有多少个待处理的请求。然而，由于sqe的寿命只是实际提交它的寿命，所以应用程序有可能开比SQ环大小更高的待处理请求数。应用程序必须注意不要这样做，否则可能会有溢出CQ环的风险。默认情况下，CQ环的大小是SQ环的两倍。这允许应用程序在管理这方面有一定的灵活性，但并没有完全消除这样做的必要性。如果应用程序确实违反了这一限制，它将被跟踪为CQ环中的溢出条件。后面会有更多细节。

完成事件可能以任何顺序到达，请求提交和关联完成之间没有顺序。SQ和CQ环相互独立运行。然而，一个完成事件将总是对应于一个给定的提交请求。因此，一个完成事件将始终与特定的提交请求相关联。

## 5.0 io_uring接口

就像 aio 一样，io_uring 有许多与之相关的系统调用来定义它的操作。第一个系统调用是一个调用来设置一个io_uring实例。
```c
int io_uring_setup(unsigned entries, struct io_uring_params *params);
```

应用程序必须为这个io_uring实例提供所需的条目数，以及与之相关的一组参数。条目表示将与这个io_uring实例相关联的sqes的数量。它必须是2的幂，范围是1～4096（两者都包含在内）。params结构可以被内核读取和写入，它的定义如下。

```c
struct io_uring_params {
    __u32 sq_entries;
    __u32 cq_entries;
    __u32 flags;
    __u32 sq_thread_cpu;
    __u32 sq_thread_idle;
    __u32 resv[5];
    struct io_sqring_offsets sq_off;
    struct io_cqring_offsets cq_off;
};
```

sq_entries 将由内核填写，让应用程序知道这个环支持多少 sqe 条目。同样，对于cqe条目，cq_entries成员告诉应用程序这个CQ环有多大。除了 sq_off 和 cq_off 字段之外，其余结构的讨论将推迟到高级用例部分，因为它们是通过 io_uring 设置基本通信所必需的。

当成功调用io_uring_setup(2)时，内核将返回一个文件描述符，用于引用这个io_uring实例。这就是sq_off和cq_off结构的用武之地。鉴于sque和cqe结构是由内核和应用程序共享的，应用程序需要一种方法来获得对这些内存的访问。这是通过mmap(2)映射到应用程序的内存空间来实现的。应用程序使用 sq_off 成员来计算不同环成员的偏移量。io_sqring_offsets结构如下。

```c
struct io_sqring_offsets {
    __u32 head; /* offset of ring head */
    __u32 tail; /* offset of ring tail */
    __u32 ring_mask; /* ring mask value */
    __u32 ring_entries; /* entries in ring */
    __u32 flags; /* ring flags */
    __u32 dropped; /* number of sqes not submitted */
    __u32 array; /* sqe index array */
    __u32 resv1;
    __u64 resv2;
};
```
要访问这个内存，应用程序必须使用io_uring文件描述符和内存偏移量来调用mmap(2)。相关联的SQ环。io_uring API定义了以下mmap偏移量，供应用程序使用。

```c
#define IORING_OFF_SQ_RING 0ULL
#define IORING_OFF_CQ_RING 0x8000000ULL
#define IORING_OFF_SQES 0x10000000ULL
```

其中IORING_OFF_SQ_RING用于将SQ环映射到应用内存空间，IORING_OFF_CQ_RING用于CQ环同上，最后IORING_OFF_SQES用于映射sqe数组。对于CQ环来说，cqes数组是CQ环本身的一部分。由于sq环是值进入sque数组的索引，所以sque数组必须由应用程序单独映射。

应用程序将定义自己的结构来存放这些偏移量。下面是一个示例。

```c
struct app_sq_ring {
    unsigned *head;
    unsigned *tail;
    unsigned *ring_mask;
    unsigned *ring_entries;
    unsigned *flags;
    unsigned *dropped;
    unsigned *array;
};
```

同时，一个初始化的示例

```c
struct app_sq_ring app_setup_sq_ring(int ring_fd, struct io_uring_params *p)
{
    struct app_sq_ring sqring;
    void *ptr;
    ptr = mmap(NULL, p→sq_off.array + p→sq_entries * sizeof(__u32),
            PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE,
            ring_fd, IORING_OFF_SQ_RING);
    sring→head = ptr + p→sq_off.head;
    sring→tail = ptr + p→sq_off.tail;
    sring→ring_mask = ptr + p→sq_off.ring_mask;
    sring→ring_entries = ptr + p→sq_off.ring_entries;
    sring→flags = ptr + p→sq_off.flags;
    sring→dropped = ptr + p→sq_off.dropped;
    sring→array = ptr + p→sq_off.array;
    return sring;
}
```

CQ环的映射与此类似，使用IORING_OFF_CQ_RING和io_cqring_offsets cq_off成员定义的偏移量。最后，使用IORING_OFF_SQES偏移来映射sque数组。由于这是可以在应用程序之间重复使用的模板代码，liburing库接口提供了一组帮助程序来以简单的方式完成设置和内存映射。关于这一点，请参见io_uring库部分。一旦所有这些都完成了，应用程序就可以通过io_uring实例进行通信了。

应用程序还需要一种方法来告诉内核，它现在已经产生了请求供它消费。这可以通过另一个系统调用来完成。

```c
int io_uring_enter(unsigned int fd, unsigned int to_submit,
                   unsigned int min_complete, unsigned int flags,
                   sigset_t sig);
```

fd指的是Ring的文件描述符，由 io_uring_setup(2) 返回。to_submit告诉内核，有最多该数量的 sqe 准备被消耗和提交，而 min_complete 则要求内核等待该数量的请求完成。有了一个既可以提交又可以等待完成的单一调用，意味着应用程序可以用一个系统调用同时提交和等待请求完成。 flags 包含修改调用行为的标志。最重要的一个是：

```c
#define IORING_ENTER_GETEVENTS (1U << 0)
```

如果在flags中设置了IORING_ENTER_GETEVENTS，那么内核将主动等待 min_complete 事件的出现。精明的读者可能会疑惑，如果我们也有 min_complet e的话我们还需要这个flag做什么。在有些情况下这种区分是很重要的，这将在后面介绍。现在，如果你希望等待完成，必须设置IORING_ENTER_GETEVENTS。

io_uring_setup(2) 将创建一个给定大小的io_uring实例。有了这个设置应用程序就可以开始填写 sqes，并使用 io_uring_enter(2) 提交它们。可以用同一个调用来等待完成，也可以在稍后的时间单独完成。除非应用程序想等待完成，否则它也可以只检查CQ环尾是否有任何事件。内核会直接修改CQ环尾，因此应用程序可以消耗完备的事件，而不需要调用io_uring_enter(2)并设置IORING_ENTER_GETEVENTS。

关于可用的命令类型和使用方法，请参见io_uring_enter(2) man页面。

## 5.1 SQE 执行顺序

通常sqe是独立使用的，也就是说，一个sqe的执行不会影响环中后续sqe项的执行和排序。这使得操作具有充分的灵活性，并使它们能够并行执行和完成，以获得最大的效率和性能。可能需要排序的一个用例是数据完整性写入。一个常见的例子是一系列的写入，然后是fsync/fdatasync。只要我们能够允许以任何顺序完成写入，我们只关心在所有写入完成后是否执行数据同步。应用程序通常会把它变成一个写和等待的操作，然后在所有的写入都被底层存储确认后再发出同步。

io_uring支持将提交侧队列排空，直到之前所有完成的操作都完成。这允许应用程序排队执行上述同步操作，并知道在之前所有命令完成之前不会开始。这可以通过在 sqe flags 字段中设置 IOSQE_IO_DRAIN 来实现。注意，这将阻塞整个提交队列。根据 io_uring 在特定应用程序中的使用方式，这可能会引入比期望的更大的管道气泡。如果这类操作很常见的话，应用程序可能需要使用独立的io_uring上下文来进行完整性写入，以便让不相关的命令有更好的同步性能。

## 5.2 SQES链

虽然IOSQE_IO_DRAIN包含了一个完整的管道屏障，但io_uring也支持更精细的sqe序列控制。链接的sqe提供了一种方法来描述大提交环内的sqe序列之间的依赖关系，其中每个sqe的执行都依赖于前一个sqe的成功完成。这种用例可能包括一系列必须按顺序执行的写入，或者可能是类似复制的操作，即从一个文件读取后，再向另一个文件写入，两个sqe的缓冲区是共享的。要利用这个特性，应用程序必须在sqe标志字段中设置IOSQE_IO_LINK。如果设置了这个标志，那么在前一个 sqe 成功完成之前，下一个 sqe 将不会被启动。如果前一个 sqe 没有完全完成，链子就会被打断，链接的 sqe 会被取消。

-ECANCELED作为错误代码。在这里，完全完成指的是请求的完全成功完成。任何错误或潜在的短读/写都会中止请求链。请求必须完全完成。

只要在flags字段中设置了IOSQE_IO_LINK，链接的sqe链就会继续。因此，链被定义为从第一个设置了IOSQE_IO_LINK的sqe开始，到随后第一个没有设置的sqe结束。支持任意长度。
链的执行独立于提交环中的其他 sqe。链是独立的执行单元，多个链可以相互并行执行和完成。这包括不属于任何链的 sqe。

## 5.3 超时

io_uring支持的大多数命令都是对数据进行操作的，或者直接如读/写操作，或者间接如fsync风格的命令，而超时命令则有些不同。IORING_OP_TIMEOUT不是工作在数据上，而是帮助控制完成环上的等待。超时命令支持两种不同的触发类型，它们可以在一个命令中一起使用。一种触发类型是经典的超时，调用者传入一个（变体的）struct timespec，其值是非零的秒或者纳秒。为了保持32位与64位应用程序和内核空间之间的兼容性，使用的类型必须是以下格式。

```c
struct __kernel_timespec {
    int64_t tv_sec;
    long long tv_nsec;
};
```

在某些时候，用户空间应该有一个符合这种描述的 timespec64 结构。如果需要定时超时，sqe addr字段必须指向这种类型的结构。一旦指定的时间过去，超时命令就会触发。

第二种触发类型是完成次数。如果使用，应将完成次数值填入sqe的offset字段中。一旦自超时命令排队以来发生了指定的完成次数，超时命令就会完成。

6.0 内存访问排序（内存屏障）
通过io_uring实例进行安全高效通信的一个重要方面是正确使用内存访问排序。详细介绍各种架构的内存访问排序超出了本文的范围。如果你喜欢使用通过liburing库暴露的简化的io_uring API，那么你可以放心地忽略这一节，而跳到liburing库部分。如果你对使用原始接口感兴趣，那么理解本节就很重要。

为了保持简单，我们将把它简化为两个简单的内存排序操作。为了保持简短，解释有些简化。

read_barrier()。在进行后续的内存读取之前，确保之前的写入是可见的。
write_barrier()：将这次写入排序在之前的写入之后。在之前的写入之后，对这次写入进行排序。

根据相关的架构，其中的一个或两个可能都是无操作的。当使用io_uring时，这并不重要。重要的是，我们在某些架构上会需要它们，因此应用程序编写者应该了解如何做到这一点。需要一个write_barrier()来确保写入的顺序。比方说，一个应用程序想要填写一个sqe，并通知内核有一个可供消费的sqe。这是一个两阶段的过程--首先填写各个sqe成员，并将sqe索引放入sqe环数组中，然后更新sqe环尾部，以告知内核有新的条目可用。在没有任何顺序暗示的情况下，处理器完全可以按照任何它认为最优化的顺序来重新安排这些写入的顺序。让我们看看下面的例子，每个数字表示一个内存操作。

```
1: sqe→opcode = IORING_OP_READV;
2: sqe→fd = fd;
3: sqe→off = 0;
4: sqe→addr = &iovec;
5: sqe→len = 1;
6: sqe→user_data = some_value;
7: sqring→tail = sqring→tail + 1;
```

这个操作其实不能保证写7的时候使sqe对内核可见，它是序列中的最后一次写。在写7之前的所有写都是可见的，这一点非常关键，否则内核可能会看到一个写了一半的sqe。从应用的角度来看，在通知内核新的Square之前，你需要一个写屏障来保证写的正确顺序。因为实际的sqe存储发生的顺序并不重要，只要它们在尾部写入之前是可见的，我们可以在写入6之后和写入7之前使用一个排序基元。因此，这个序列看起来像下面这样。

```
 1: sqe→opcode = IORING_OP_READV;
2: sqe→fd = fd;
3: sqe→off = 0;
4: sqe→addr = &iovec;
5: sqe→len = 1;
6: sqe→user_data = some_value;
 write_barrier(); /* ensure previous writes are seen before tail write */
7: sqring→tail = sqring→tail + 1;
 write_barrier(); /* ensure tail write is seen */
```

内核会在读取SQ环的最后包含一个read_barrier()，以确保应用程序的最后的写入是可见的。从CQ环侧来看，由于消费者/生产者的角色是相反的，应用程序只需要在读取CQ环队尾之前发出一个read_barrier()，以确保它能看到内核所做的任何写入。

虽然内存排序类型已经浓缩为两种特定的类型，但根据代码运行在什么机器上，架构实现当然会有所不同。即使应用程序直接使用io_uring接口（而不是liburing助手），它仍然需要特定架构的屏障类型。liburing库提供了这些定义，建议使用应用程序中的这些定义。

有了这个关于内存排序的基本解释，以及liburing提供的帮助程序，请回过头来阅读之前引用read_barrier()和write_barrier()的例子。如果它们之前没有完全理解，希望现在能够理解了。

## 7 liburing

io_uring的内部细节讲完了，现在松一口气，来看看如何用简单的方法来完成上面的事情。liburing库有两个作用。

* 消除设置io_uring对样板代码（boilerplate code）的需求。
* 为基本用例提供简化的API。

后者保证了应用程序根本不用担心内存排序，也不用自己做任何环形缓冲区管理。这使得API的使用和理解变得更加简单。同时也消除了理解所有工作细节的需要。如果我们只专注于提供基于liburing的例子，这篇文章可能会短得多，但至少应该对内部工作原理有一定的了解以从应用程序中提取最大的性能，这通常是有益的。此外，liburing目前专注于减少样板代码并为标准用例提供基本的帮助程序。一些更高级的功能还不能通过liburing来实现。然而，这并不意味着你不能将两者混搭在一起。毕竟它们都运行着同样的代码。一般鼓励应用程序使用liburing的设置助手，即使他们同时也使用原始接口。

### 7.1 LIBURING IO_URING 初始化

让我们从一个例子开始。与其手动调用io_uring_setup(2)，然后对三个必要的区域进行mmap(2)，liburing提供了下面的基本助手来完成同样的任务。

```c
struct io_uring ring;
io_uring_queue_init(ENTRIES, &ring, 0);
```

io_uring结构保存了SQ和CQ环的信息，io_uring_queue_init(3)调用为你处理所有的设置逻辑。在这个特殊的例子中，我们将flags参数传入0。一旦一个应用程序使用io_uring实例完成，它就会简单地调用:

```c
 io_uring_queue_exit(&ring);
```

来销毁它。类似于应用程序分配的其他资源，一旦应用程序退出，它们就会被内核自动回收。对于应用程序可能创建的任何io_uring实例也是如此。

### 7.2  LIBURING 提交与完成

一个非常基本的用例是提交一个请求，然后等待它的完成。有了 liburing 助手，代码如下：
```c
struct io_uring_sqe sqe;
struct io_uring_cqe cqe;
/* get an sqe and fill in a READV operation */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_readv(sqe, fd, &iovec, 1, offset);
/* tell the kernel we have an sqe ready for consumption */
io_uring_submit(&ring);
/* wait for the sqe to complete */
io_uring_wait_cqe(&ring, &cqe);
/* read and process cqe event */
app_handle_cqe(cqe);
io_uring_cqe_seen(&ring, cqe);
```

这个应该很容易看明白。最后一次调用io_uring_wait_cqe(3)将返回我们刚刚提交的sqe的完成事件，前提是你没有其他sqe正在处理。如果有那么完成事件可能是另一个sqe的。

如果应用程序仅仅希望监听完成的消息而不是等待事件可用，可以使用io_uring_peek_cqe(3)。对于这两种用例，应用程序必须在完成这个完成事件后调用io_uring_cqe_seen(3)。或者反复调用io_uring_peek_cqe(3)或io_uring_wait_cqe(3)则会一直返回相同的事件。io_uring_cqe_seen(3)会调整CQ环首部指针，使内核能够在同一槽位上填入新的事件。

有各种填入sqe的帮助程序，io_uring_prep_readv(3)只是其中一个例子。我鼓励应用程序总是尽可能地利用liburing提供的帮助程序。

liburing库还处于起步阶段，并且还在不断地开发，以扩展支持的功能和可用的帮助程序。

## 8.0 高级用户功能与特性

上面的例子和用例适用于各种类型的IO，无论是基于O_DIRECT文件的IO，还是缓冲IO，套接字IO等等。不需要特别注意保证它们的正确操作或异步性质。然而，io_uring确实提供了一些应用程序需要选择的功能。下面的小节将描述其中的大部分。

### 8.1 固定文件与缓冲

每次当一个文件描述符被填入sqe并提交给内核时，内核必须检索到所述文件的引用。一旦IO完成，文件引用又会被丢弃。由于这个文件引用的原子性，对于高IOPS的工作负载来说，这可能是一个明显的减速。为了缓解这个问题，io_uring提供了一种为io_uring实例预先注册文件集的方法。这是通过系统调用完成的。

```c
int io_uring_register(unsigned int fd, unsigned int opcode, void *arg, unsigned int nr_args);
```

fd是io_uring实例环文件描述符，opcode是指正在进行的注册类型。为了注册一个文件集合，必须使用IORING_REGISTER_FILES，然后args必须指向一个应用程序已经打开的文件描述符数组，nr_args必须包含数组的大小。一旦io_uring_register(2)成功地完成了一个文件集的注册，应用程序就可以通过将数组中的文件描述符的索引（而不是实际的文件描述符）分配给 sqe→fd 字段来使用这些文件，并通过在 sqe→flags 字段中设置IOSQE_FIXED_FILE将其标记为一个文件集fd。通过将 sqe→flags 设置为非注册的fd，而不在flags中设置IOSQE_FIXED_FILE，即使注册了一个文件集，应用程序也可以自由地继续使用非注册的文件。当io_uring实例被销毁时，文件集合被自动销毁，也可以通过在io_uring_register(2)的操作码中使用IORING_UNREGISTER_FILES手动释放。

也可以注册一组固定的IO缓冲区。当使用O_DIRECT时，内核必须在对应用页面进行IO之前将其映射到内核中，并在IO完成后取消映射这些页面。这可能是一个昂贵的操作。如果一个应用程序重复使用IO缓冲区，那么可以只需要一次映射和解映射。要为IO注册一组固定的缓冲区，必须调用io_uring_register(2)，操作码为IORING_REGISTER_BUFFERS，然后args必须包含一个填入了地址和长度的iovec结构的数组。在成功注册缓冲区后，应用程序可以使用IORING_OP_READ_FIXED和IORING_OP_WRITE_FIXED来执行与这些缓冲区的IO。当使用这些固定的操作码时，sqe→addr 必须包含一个在这些缓冲区中的地址，sqe→len 必须包含请求的长度（以字节为单位）。应用程序可以注册比任何给定IO操作更大的缓冲区，固定读写只是单个固定缓冲区的子集是完全合法的。

### 8.2 IO池
对于追求最低延迟的应用，io_uring提供了对文件轮询IO的支持。在这种情况下，轮询指的是在不依赖硬件中断信号完成事件的情况下执行IO。当IO被轮询时，应用程序将反复向硬件驱动程序询问提交的IO请求的状态。这与非轮询IO不同，在非轮询IO中，应用程序通常会进入睡眠状态，等待ardware中断作为其唤醒源。对于极低延迟的设备，轮询可以显著提高性能。对于非常高IOPS的应用也是如此，高中断率使得非轮询负载有更高的开销。在延迟或总体IOPS率方面，轮询何时是有意义的，根据应用、IO设备和机器的能力而有所不同。

要利用IO轮询，必须在传递给io_uring_setup(2)系统调用或io_uring_queue_init(3) liburing 库帮助程序的标志中设置IORING_SETUP_IOPOLL。当利用轮询时，应用程序不能再检查CQ环尾是否有完成，因为不会有一个自动触发的异步硬件侧完成事件。取而代之的是，应用程序必须通过调用io_uring_enter(2)，并设置IORING_ENTER_GETEVENTS和min_complete设置为所需的事件数来主动查找和捕获这些事件。对于轮询IO来说，这要求内核简单地检查驱动侧的完成事件，而不是不断地循环这样做。

只有对轮询完成有意义的操作码才可以用在用IORING_SETUP_IOPOLL注册的io_uring实例上。这些包括任何一个读/写命令。IORING_OP_READV, IORING_OP_WRITEV, IORING_OP_READ_FIXED, IORING_OP_WRITE_FIXED. 在注册了轮询的 io_uring 实例上发布一个不可轮询的 op 代码是非法的。这样做会导致 io_uring_enter(2) 的 -EINVAL 返回。原因是内核无法知道在设置了 IORING_ENTER_GETEVENTS 的情况下对 io_uring_enter(2) 的调用是可以安全地等待事件回调还是应该主动地进行轮询。

尽管io_uring一般来说效率较高，可以让更多的请求通过较少的系统调用同时发出和完成，但在某些情况下，我们仍然可以通过进一步减少执行IO所需的系统调用次数来提高效率。内核侧轮询（kernel side polling）就是这样一个功能。启用后，应用程序不再需要调用io_uring_enter(2)来提交IO。当应用程序更新SQ环并填写一个新的sqe时，内核侧会自动注意到新的条目（或条目）并提交它们。这是通过一个内核线程完成的，具体到某一条具体的io_uring。

要使用这个功能，io_uring实例必须在io_uring_params成员中注册IORING_SETUP_SQPOLL，或者传递给io_uring_queue_init(3)。此外，如果应用程序希望将这个线程限制在一个特定的CPU上，也可以通过标记IORING_SETUP_SQ_AFF来实现，并将io_uring_params sq_thread_cpu设置为所需的CPU。需要注意的是，用IORING_SETUP_SQPOLL设置io_uring实例是一个特权操作。如果用户没有足够的权限，io_uring_queue_init(3)将会报告-EPERM失败。

为了避免在io_uring实例处于非活动状态时浪费太多CPU，内核侧线程会在空闲一段时间后自动进入SLEEP状态。当这种情况发生时，线程会在SQ环标志成员中设置IORING_SQ_NEED_WAKEUP。当设置了这一点后，应用程序就不能依靠内核自动寻找新的条目，它必须在设置了IORING_ENTER_SQ_WAKEUP的情况下再调用io_uring_enter(2)。应用端逻辑通常是这样的：

```c
/* fills in new sqe entries */
add_more_io();
/*
* need to call io_uring_enter() to make the kernel notice the new IO
* if polled and the thread is now sleeping.
*/
if ((*sqring→flags) & IORING_SQ_NEED_WAKEUP)
io_uring_enter(ring_fd, to_submit, to_wait, IORING_ENTER_SQ_WAKEUP);
```

只要应用程序一直使用IO，IORING_SQ_NEED_WAKEUP就永远不会被设置，我们可以有效地执行IO，而不用执行一次系统调用。不过，在应用程序中要始终保持类似于上述的逻辑以防线程真的进入睡眠状态。具体的空闲前的宽限期可以通过设置io_uring_params sq_thread_idle成员来配置。这个值的单位是毫秒。如果这个成员没有被设置，内核默认在一秒钟空闲后进入睡眠状态。

对于 "正常的 "IRQ驱动的IO，完成事件可以通过直接在应用程序中查看CQ环来发现。如果io_uring实例是用IORING_SETUP_IOPOLL设置的，那么内核线程也会负责处理完成事件。因此，对于这两种情况，除非应用程序想等待IO的发生，否则它可以简单地监视CQ环来寻找完成事件。

## 9.0 性能
最后，io_uring达到了为它设定的设计目标。我们在内核和应用程序之间有一个非常高效的传递机制，以两个不同的环的形式存在。虽然原始的接口在应用程序中的正确使用需要一些注意，但主要的复杂问题主要需要明确的内存排序。这些被归结为在提交和完成方面发出和处理事件的一些具体细节，并且一般会在不同的应用程序中遵循相同的模式。随着 liburing 接口的不断成熟，我预计大多数的应用使用liburing的API就会足够满意了。

虽然本说明的目的不是要全面详细地介绍io_uring所实现的性能和可扩展性，但本节将简要地谈谈在这一领域观察到的一些成果。更多细节请参见[1]。请注意，由于SSD存储器方面的进一步改进，这些结果有点过时了。例如，在我的测试机上，使用io_uring的每核性能峰值现在大约是1700K 4k IOPS，而不是1620K。请注意这些数值并没有太多绝对意义，它们主要是在衡量相对改进方面有用。我们将继续通过使用io_uring找到更低的延迟和更高的峰值性能，现在应用程序和内核之间的通信机制不再是瓶颈。

### 9.1 裸性能
观察接口的原始性能有很多方法。大多数测试也会涉及到内核的其他部分。上文中的数字就是这样一个例子，我们通过随机从块设备或文件中读取数据来衡量性能。对于峰值性能来说，io_uring帮助我们在轮询的情况下达到了1.7M 4k IOPS。 aio达到的性能瓶颈比这个低得多，为608K。这里的比较不是很公平，因为aio不支持轮询IO。如果我们禁用轮询，io_uring能够在（其他情况下）相同的测试情况下驱动大约120万IOPS。在这一点上，aio的局限性是相当明显的，io_uring在相同的工作负载下，可以驱动两倍的IOPS量。

io_uring也支持no-op命令，它主要用于检查接口的原始吞吐量。根据所使用的系统，从每秒12M消息（我的笔记本电脑）到每秒20M消息（用于其他引用结果的测试机）都有可能。实际的结果根据具体的测试机变化很大，主要是受系统调用次数的约束。原始接口的速度本来就受内存性能的约束，而提交消息和完成消息在内存中的占比都很小，而且是线性的，所以实现的消息每秒速率可以很高。

### 9.2 带缓冲的异步操作性能
我之前提到过，内核内缓冲的AIO实现可能比在用户空间完成的更有效率。一个主要的原因是与缓存与非缓存数据有关。在做缓冲IO的时候，应用一般会严重依赖内核的页面缓存来获得良好的性能。用户空间的应用无法知道它接下来要请求的数据是否被缓存。当然它可以查询这些信息，但这需要更多的系统调用，而且答案总是很生硬的——这一刻被缓存的数据可能在几毫秒后失效了。因此，一个有IO线程池的应用总是要把请求弹回到async上下文，导致至少有两次上下文切换。如果请求的数据已经在页面缓存中，就会导致性能急剧下降。

io_uring会像处理其他有可能阻塞应用的资源一样处理这种情况。更重要的是，对于那些不会阻塞的操作，数据是同步操作的。这使得io_uring对于已经在页面缓存中的IO和普通同步接口一样高效。一旦IO提交调用返回，应用程序在CQ环中已经有一个完成事件在等待，数据也已经被复制。

## 10.0 进一步阅读
鉴于这是一个全新的接口，我们还没有得到很多人的采用。在写这篇文章的时候，带有这个接口的内核还在RC阶段。即使有了一个相当完整的接口描述，研究利用io_uring的程序对于充分理解如何最好地使用它也是有好处的。

其中一个例子是fio[2]中的io_uring引擎。除了注册文件集之外，它也能够使用所有描述的高级功能。
另一个例子是 t/io_uring.c 样例基准程序，它也是fio附带的。它只是简单地对文件或设备进行随机读取，通过可配置的设置来探索高级用例的整个功能集。
liburing[3] 库有一套完整的系统调用接口的man页值得一读。它还附带了一些测试程序，既有针对开发过程中发现的问题的单元测试，也有技术演示。
LWN还写了一篇很好的文章[4]，介绍了io_uring的早期阶段。需要注意的是，在这篇文章写完之后，io_uring做了一些改动，因此我建议在两者有差异的情况下，以这篇文章为准。

## 11.0 引用
[1] https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/
[2] git://git.kernel.dk/fio
[3] git://git.kernel.dk/liburing
[4] https://lwn.net/Articles/776703/
Version: 0.4, 2019-10-15