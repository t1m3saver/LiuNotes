# UNIX环境高级编程学习笔记

## 1、unix和linux的关系

Unix 和 Linux 的关系可以用一句话概括：**Linux 是一个“类 Unix”系统，它继承了 Unix 的设计思想，但并不是 Unix 的直接后代。Linux 是受 Unix 启发并高度兼容 POSIX 的自由开源系统，是 Unix 精神的继承者，而非其技术克隆**

## 2、那啥是POSIX呢

> **POSIX（Portable Operating System Interface）是一套操作系统接口标准，规定了“像 Unix 那样”的行为方式。**

说白了，就是一种标准

## 3、口令文件/etc/passwd

| **字段位置** | **含义**                           |
| ------------ | ---------------------------------- |
| 1            | 用户名（如 root）                  |
| 2            | 密码占位符（以前是明文，现在是 x） |
| 3            | UID（用户ID）                      |
| 4            | GID（组ID）                        |
| 5            | 用户描述（备注）                   |
| 6            | 主目录路径                         |
| 7            | 默认 shell                         |

现在密码信息已经被分离出来，放到 /etc/shadow 文件中。

**这个文件权限严格受限**，通常只有 root 可读

## 4、标准输入、输出、错误，其实都是文件描述符fd

- 当你 printf("hello") 或 cout << "hello"; 的时候，

  

  - 底层是调用 write(1, "hello", 5)，
  - 操作系统把数据写到了 /dev/tty 这个设备文件上，
  - 然后你在屏幕上看到了“hello”

也就是说，也是在写文件呢。贯彻了一切都是在写文件

## 5、/dev下的设备文件是操作系统为设备驱动暴露出来的访问接口

它本身只是个门牌号（metadata + 一个 major:minor 号），真正干活的是后面的**驱动程序**。



**不是磁盘，不是屏幕，不是键盘自己管理自己，是驱动在管。**



所以：



- /dev/tty -> 是**终端设备驱动**在管理。
- /dev/sda -> 是**磁盘设备驱动**在管理。
- /dev/fb0 -> 是**显存framebuffer驱动**在管理。
- /dev/input/event0 -> 是**输入设备驱动**（键盘、鼠标）在管理。

设备文件本身什么都不做，是内核拿它**做跳板**，调到对应设备驱动的操作函数（read、write、ioctl、mmap…）。

## 6、**就算你只写了1字节，磁盘实际上也要按”块”单位读-改-写一整块。**

| **原因**     | **解释**                                                     |
| ------------ | ------------------------------------------------------------ |
| 硬件物理限制 | 磁盘、固态硬盘，物理上只能按固定大小（比如512B, 4KB）操作。  |
| 效率与寿命   | 频繁写1字节会严重降低硬盘/SSD寿命，按块写可以均衡负载、延长寿命。 |
| 文件系统设计 | Ext4、XFS、NTFS等文件系统都是基于块的抽象和管理来的。        |

- 把对应的整个块（比如4KB）读进内存
- 修改其中1字节
- 把整块重新写回磁盘

这叫 **读-改-写**（read-modify-write）过程。

## 7、为什么磁盘需要先读了再写

想象一下极端情况：



- 第100字节要写字母’A’。
- 但这100字节属于第3个块（比如第3个4KB块）。
- 如果你直接write(1 byte)，你只能提交这1 byte数据给磁盘。
- 磁盘内部要求是**一整块**（4096字节），你只给了1字节，剩下的4095字节它拿什么填？
- 结果就是：**剩余部分会变成随机垃圾或者清零。**



换句话说：



> 你如果不先读回来整个块的原始数据，就无法保证**修改后的块**仍然是正确的。

比如数据库系统（MySQL, RocksDB）经常这么干。



> **只要你覆盖了整个块，内核就可以直接write，不用r-m-w。**



所以性能优化经常讲：



- 尽量按块对齐写入。
- 尽量写大块数据，避免碎片小写。



## 8、内核通过7个函数将一个可执行文件载入内容中

## 9、进程控制的核心三函数：exec、fork、waitpid

## 10、ctrl d是默认的文件结束符

## 11、“一个进程内的所有线程共享同一地址空间、文件描述符、栈以及与进程相关的属性”

## 12、组文件/etc/group

包含了组名和组id的映射关系

## 13、getuid与getgid分别是用户id和组id

## 14、一个用户至少支持16个附属组

## 15、信号处理有三种方式：按系统默认、忽略信号、提供处理函数

## 16、ctrl + \表示退出

## 17、向一个进场发送信号的时候，要么是所有者要么是root

## 18、3个时间

“当度量一个进程的执行时间时（见3.9节），UNIX系统为一个进程维护了3个进程时间值：
•时钟时间；
•用户CPU时间；
•系统CPU时间。
时钟时间又称为墙上时钟时间（wall clock time），它是进程运行的时间总量，其值与系统中同时运行的进程数有关。每当在本书中提到时钟时间时，都是在系统中没有其他活动时进行度量的。
用户CPU时间是执行用户指令所用的时间量。系统CPU时间是为该进程执行内核程序所经历的时间。例如，每当一个进程执行一个系统服务时，如read或write，在内核内执行该服务所花费的时间就计入该进程的系统CPU时间。用户CPU时间和系统CPU时间之和常被称为CPU时间。
要取得任一进程的时钟时间、用户时间和系统时间是很容易的—只要执行命令 time(1)，其参数是要度量其执行时间的命令”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 19、“Linux 3.2.0提供了380个系统调用，FreeBSD 8.0提供的系统调用超过450个。”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 20、“文件描述符的变化范围是0～OPEN_MAX-1（见图2-11）。早期的UNIX系统实现采用的上限值是19（允许每个进程最多打开20个文件），但现在很多系统将其上限值增加至63。”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 21、文件描述符

“文件描述符（file descriptor）通常是一个小的非负整数，内核用以标识一个特定进程正在访问的文件。当内核打开一个现有文件或创建一个新文件时，它都返回一个文件描述符。在读、写文件时，可以使用这个文件描述符。”“按惯例，每当运行一个新程序时，所有的 shell 都为其打开 3 个文件描述符，即标准输入（standard input）、标准输出（standard output）以及标准错误（standard error）。如果不做特殊处理，例如就像简单的命令ls，则这3个描述符都链接向终端。大多数shell都提供一种方法，使其中任何一个或所有这3个描述符都能重新定向到某个文件”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 22、<unistd.h>、STDIN_FILENO、STDOUT_FILENO

“头文件<unistd.h>（apue.h中包含了此头文件）及两个常量STDIN_FILENO和STDOUT_FILENO是POSIX标准的一部分（下一章将对此做更多的说明）。头文件<unistd.h>包含了很多UNIX系统服务的函数原型，例如图1-4程序中调用的read和write。
两个常量STDIN_FILENO和STDOUT_FILENO定义在<unistd.h>头文件中，它们指定了标准输入和标准输出的文件描述符。在POSIX标准中，它们的值分别是0和1，但是考虑到可读性，我们将使用这些名字来表示这些常量”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 23、fork、exec、waitpid

“有3个用于进程控制的主要函数：fork、exec和waitpid。（exec函数有7种变体，但经常把它们统称为exec函数。）”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 24、fork

“调用fork创建一个新进程。新进程是调用进程的一个副本，我们称调用进程为父进程，新创建的进程为子进程。fork对父进程返回新的子进程的进程ID（一个非负整数），对子进程则返回0。因为fork 创建一个新进程，所以说它被调用一次（由父进程），但返回两次（分别在父进程中和在子进程中）。”

“在子进程中，调用 execlp 以执行从标准输入读入的命令。这就用新的程序文件替换了子进程原先执行的程序文件。fork和跟随其后的exec两者的组合就是某些操作系统所称的产生（spawn）一个新进程”

“子进程调用 execlp 执行新程序文件，而父进程希望等待子进程终止，这是通过调用waitpid实现的，其参数指定要等待的进程（即pid参数是子进程ID）。waitpid函数返回子进程的终止状态（status 变量）。在我们这个简单的程序中，没有使用该值。”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。



## 25、线程和线程id

“通常，一个进程只有一个控制线程（thread）—某一时刻执行的一组机器指令。对于某些问题，如果有多个控制线程分别作用于它的不同部分，那么解决起来就容易得多。另外，多个控制线程也可以充分利用多处理器系统的并行能力。
一个进程内的所有线程共享同一地址空间、文件描述符、栈以及与进程相关的属性。因为它们能访问同一存储区，所以各线程在访问共享数据时需要采取同步措施以避免不一致性。
与进程相同，线程也用ID标识。但是，线程ID只在它所属的进程内起作用。一个进程中的线程 ID 在另一个进程中没有意义。当在一进程中对某个特定线程进行处理时，我们可以使用该线程的ID引用它。
控制线程的函数与控制进程的函数类似，但另有一套。线程模型是在进程模型建立很久之后才被引入到UNIX系统中的”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 26、c标准的两个错误打印函数

“C标准定义了两个函数，它们用于打印出错信息。
#include <string.h>
char *strerror(int errnum);”

“strerror函数将errnum（通常就是errno值）映射为一个出错消息字符串，并且返回此字符串的指针。
perror函数基于errno的当前值，在标准错误上产生一条出错消息，然后返回。
#include <stdio.h>
void perror(const char *msg);
它首先输出由msg指向的字符串，然后是一个冒号，一个空格，接着是对应于errno值的出错消息，最后是一个换行符。”

## 27、致命错误和非致命错误

“可将在<errno.h>中定义的各种出错分成两类：致命性的和非致命性的。对于致命性的错误，无法执行恢复动作。最多能做的是在用户屏幕上打印出一条出错消息或者将一条出错消息写入日志文件中，然后退出。对于非致命性的出错，有时可以较妥善地进行处理。大多数非致命性出错是暂时的（如资源短缺），当系统中的活动较少时，这种出错很可能不会发生。
与资源相关的非致命性出错包括：EAGAIN、ENFILE、ENOBUFS、ENOLCK、ENOSPC、EWOULDBLOCK，有时ENOMEM也是非致命性出错。当EBUSY指明共享资源正在使用时，也可将它作为非致命性出错处理。当 EINTR 中断一个慢速系统调用时，可将它作为非致命性出错处理（在10.5节对此会进行更多说明）。
对于资源相关的非致命性出错的典型恢复操作是延迟一段时间，然后重试。这种技术可应用于其他情况。例如，假设出错表明一个网络连接不再起作用，那么应用[…]”

摘录来自
UNIX环境高级编程(第3版)
[美]W. Richard Stevens Stephen A. Rago 著
此材料可能受版权保护。

## 28、内核如何使用用户 ID 来检验该用户是否有执行某些操作的权限

“用户 ID 为 0 的用户为根用户（root）或超级用户（superuser）。在口令文件中，通常有一个登录项，其登录名为 root，我们称这种用户的特权为超级用户特权。我们将在第 4 章中看到，如果一个进程具有超级用户特权，则大多数文件权限检查都不再进行。某些操作系统功能只向超级用户提供，超级用户对系统有自由的支配权。
Mac OS X客户端版本交由用户使用时，禁用超级用户账户，服务器版本则可使用该账户。在Apple 的网站可以找到使用说明，它告知如何才能使用该账户”

## 29、/etc/group

“口令文件登录项也包括用户的组ID（group ID），它是一个数值。组ID也是由系统管理员在指定用户登录名时分配的。一般来说，在口令文件中有多个登录项具有相同的组 ID。组被用于将若干用户集合到项目或部门中去。这种机制允许同组的各个成员之间共享资源（如文件）。4.5 节将介绍可以通过设置文件的权限使组内所有成员都能访问该文件，而组外用户不能访问。
组文件将组名映射为数值的组ID。组文件通常是/etc/group。”

“对于磁盘上的每个文件，文件系统都存储该文件所有者的用户ID和组ID。存储这两个值只需4个字节”

“但是对于用户而言，使用名字比使用数值方便，所以**口令文件**包含了登录名和用户 ID 之间的映射关系，而**组文件**则包含了组名和组ID之间的映射关系。例如，ls -l命令使用口令文件将数值的用户ID映射为登录名，从而打印出文件所有者的登录名。
早期的UNIX系统使用16位整型数表示用户ID和组ID。现今的UNIX系统使用32位整型数表示用户ID和组ID”

“除了在口令文件中对一个登录名指定一个组ID外，大多数 UNIX系统版本还允许一个用户属于另外一些组。这一功能是从4.2BSD开始的，它允许一个用户属于多至16个其他的组。登录时，读文件/etc/group，寻找列有该用户作为其成员的前 16 个记录项就可以得到该用户的附属组ID（supplementary group ID）。在下一章将说明，POSIX要求系统至少应支持8个附属组，实际上大多数系统至少支持16个附属组。”

## 30、信号“信号（signal）用于通知进程发生了某种情况”

“进程有以下3种处理信号的方式。
（1）忽略信号。有些信号表示硬件异常，例如，除以0或访问进程地址空间以外的存储单元等，因为这些异常产生的后果不确定，所以不推荐使用这种处理方式。
（2）按系统默认方式处理。对于除数为0，系统默认方式是终止该进程。
（3）提供一个函数，信号发生时调用该函数，这被称为捕捉该信号。通过提供自编的函数，我们就能知道什么时候产生了信号，并按期望的方式处理它。
很多情况都会产生信号。终端键盘上有两种产生信号的方法，分别称为中断键（interrupt key，通常是Delete键或Ctrl+C）和退出键（quit key，通常是Ctrl+\），它们被用于中断当前运行的进程。另一种产生信号的方法是调用**kill**函数。在一个进程中调用此函数就可向另一个进程发送一个信号。当然这样做也有些限制：当向一个进程发送信号时，我们必须是“那个进程的所有者或者是超级用户。”



## 31、日历时间和进程时间

“1）日历时间。该值是自协调世界时（Coordinated Universal Time，UTC）1970年1月1日00:00:00这个特定时间以来所经过的秒数累计值（早期的手册称UTC为格林尼治标准时间）。这些时间值可用于记录文件最近一次的修改时间等。
系统基本数据类型time_t用于保存这种时间值。
（2）进程时间。也被称为CPU时间，用以度量进程使用的中央处理器资源。进程时间以时钟滴答计算。每秒钟曾经取为50、60或100个时钟滴答。
系统基本数据类型clock_t保存这种时间值”



“当度量一个进程的执行时间时（见3.9节），UNIX系统为一个进程维护了3个进程时间值：
•**时钟时间**；
•**用户CPU时间**；
•**系统CPU时间**。

时钟时间又称为墙上时钟时间（wall clock time），它是**进程运行的时间总量**，其值**与系统中同时运行的进程数有关**。每当在本书中提到时钟时间时，都是在系统中没有其他活动时进行度量的。

用**户CPU时间是执行用户指令所用的时间量**。 

**系统CPU时间是为该进程执行内核程序所经历的时间**。



例如，每当一个进程执行一个系统服务时，如read或write，在内核内执行该服务所花费的时间就计入该进程的系统CPU时间。用户CPU时间和系统CPU时间之和常被称为CPU时间。
要取得任一进程的时钟时间、用户时间和系统时间是很容易的—只要执行命令 time(1)，其参数是要度量其执行时间的命令，例如



