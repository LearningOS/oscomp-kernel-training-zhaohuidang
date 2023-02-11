# syscall/API 实现

## fs

### 轮询

#### 简介

轮询系列API的用途是轮询文件(主要是抽象的特殊文件,如流,硬件,pipe和socket),我们实现的共计4个,pselect, select, ppoll, poll; select和poll二者分别是"与"和"或"的关系: 如果要全部被轮询的文件都有待等待时间发生才返回,用select, 否则用poll(有一个发生就返回)  
实现目标上,轮询系列API包含select和poll系列,考虑到musl中普遍使用将sigmask置为空的带p的api代替原api,不妨直接实现p版api.  

带有p的函数提供了更大的灵活性,其允许附加一个sigmask从而在轮询过程中屏蔽特定的信号(当然,如果你实现的轮询是完全不允许中间插入信号的执行的,这个功能相当于没有用)

#### ppoll

`poll`(轮询)的api,功能是检查特定的事件,如准备好输入/输出,更新等是否对特定的文件已经发生
函数共有2个,分别为`ppoll`和`poll`,Linux 标准的ppoll()有三个参数:`PollFd`\*用于储存轮询内容数组, `ndst`为需要检测的文件数量, `timespec`用于标记timeout, 超过该时间就退出,如果指针为空就阻塞等待, 另外,上述除了数量传入的都是指针,也就是说,完整的实现还依赖于将用户空间的所有的数据结构内容搬入内核空间
ppoll和poll的关系类似于pselect和select的关系(manpage原话),前者是包括对信号的阻塞(可以选择暂时禁用某些信号)和更高的时间精度的升级版本 也就是多了一个信号阻塞的功能(一开始我一看到"signal"就打退堂鼓,因为要实现完整的poll功能还需要一个信号机制的实现,但是仔细翻看手册之后,发现其实如果进程调度中根本不实现信号机制,ppoll就和poll一样)
##### 基本环境
read,和所有的使用stdin的rCore的API一样,使用原版RISC-V SBI所带的sbi\_console\_getchar,其为非阻塞的api,也就是说,如果read如果要实现成阻塞,则需要自行忙等待,而且其没有外在的异步的检测完成或者Buffer的api,所以无法直接实现poll中检查是否有输入,只能设置缓冲区进行getchar尝试
##### 暂时绕过的内容
出于聚焦核心功能的目的,暂时将参数hard coded到stdin上,不进行用户空间的访存
##### 核心等待功能的实现
曾经,我们模仿之前read的读入/读出,如果没读到,就切换进程,但发现如果这么干,就会导致其输入不了任何的数据,可能(当然也可能我的理解有问题)是因为其read是不阻塞的,如果不一直检查,就会漏掉输入的情况
另外,由于Unix/Linux的文件隐喻思想,文件实际上是保罗万象的通用对象,其所有的文件相关的内容,包括read/write和各种状态,如wait都需要自己实现.
另外,rCore的read实际上不是标准的read: 标准的read是可以设置成一次性输入多个或者一个字符的,但是原版的read实质上是一个getchar. 所以,我们修改了user_lib中的read并添加了getchar, 改为允许读入多个字符的版本. 观察具体的stdin的数据结构,发现其含有一个Mutex包裹的整数,刚好可以用来作输入数据的缓存,然后每次先getchar读入一个字符到缓存,等到需要检测ready与否的时候就看缓冲区是否为空,如果空,再读一个进缓冲区,看第二次的结果,而读取的时候则先读缓冲区再读字符流就行。

#### pselect

